## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's CSR-based registration mechanism requires the cluster's Kubernetes API server to support issuing client certificates. This is discouraged on some  Kubernetes environment, and some operators of Azure Kubernetes Service (AKS) hubs and managed clusters are under a compliance requirement that forbids storing any long-lived credential - client certificate, token, or secret - inside a cluster at all. This enhancement adds an Azure AD (Entra ID) based registration mechanism so a managed cluster can join a hub using an Azure AD access token, obtained through Managed Identity, an environment-supplied credential, or Workload Identity Federation.

## Motivation

Cluster administrators increasingly prefer a managed Kubernetes service (AKS, EKS, GKE, etc.) over
self-managing the control plane. On AKS specifically, some registration flows benefit from avoiding
client-certificate issuance altogether: certificate lifecycle (issuance, rotation, storage) is one
more thing to secure and audit, and organizations with a strict no-stored-secrets policy need a way to
authenticate that never persists credential material on the cluster in the first place.

Azure provides three standard ways for a workload to obtain an Azure AD identity: Managed Identity
(the identity already attached to the underlying node, no separate setup needed), an
environment-supplied service-principal credential for cases where credential provisioning is already
handled by an external pipeline, and Workload Identity Federation (a short-lived, automatically-rotated
token tied to a specific Kubernetes ServiceAccount, with nothing persisted anywhere). None of these are
usable for OCM registration today.

### Goals

- Let a managed cluster register to a hub using an Azure AD identity.
- Support all three Azure credential mechanisms above through the same driver, with the operator
  selecting explicitly which one applies (`azure.credential`) rather than an automatic runtime
  fallback - required so the Helm chart can expose four distinct, validated configuration shapes
  (Managed Identity, Workload Identity Federation, service-principal secret, service-principal
  certificate) instead of one chart input path that behaves differently depending on which
  environment variables happen to be present at runtime.
- Make the authentication strategy pluggable at the config level, consistent with how OCM already
  supports multiple registration strategies (`csr`, gRPC, and one other cloud-IAM-based strategy
  today).
- Grant an Azure-authenticated cluster the same hub-side permissions a CSR-authenticated cluster
  already gets, through the existing permission model - no new permission surface.
- Work correctly across all four klusterlet deployment shapes: `Default`, `Singleton`, `Hosted`, and
  `SingletonHosted`.
- `clusteradm` tooling provides options to select `azure` registration while initializing the hub as
  well as joining a managed cluster to the hub.

### Non-Goals

- Support for addons authenticating to the hub using this same Azure AD identity. Addons continue to
  use the existing `token`/`csr` addon drivers regardless of which driver the managed cluster itself
  joined with.
- Any change to the `RegisterDriver` or `HubDriver` interfaces themselves. This is a new
  implementation of both, not a change to their shape.
- A generic multi-cloud identity abstraction spanning more than one cloud provider in a single driver.
- Any dynamic creation of Azure cloud resources (identities, role assignments, federated credentials)
  by the hub controller. Unlike a model where the hub provisions cloud-provider identities on demand,
  the Azure AD identity and its trust relationship are established out-of-band by whoever administers
  the Azure AD tenant, before a managed cluster ever attempts to join. The hub controller's role is
  limited to Kubernetes-native RBAC.

## Proposal

### User Stories

#### Story 1 - Hub administrator initializes a hub cluster using Azure AD authentication strategy

It must be possible for the hub administrator to specify they wish to authenticate registration
requests using the `azure` authentication strategy in the `clusteradm init` command. The default
authentication strategy remains `csr`:

```
% clusteradm init \
     --wait \
     --registration-auth=azure \
     --context ${CTX_HUB_CLUSTER}
```

This enables the driver hub-wide; it does not name any specific managed cluster's identity. The
only azure-specific hub configuration is optional and applies to every joining cluster equally: a
list of regex patterns matched against a joining cluster's Azure AD object ID for auto-approval
(`--auto-approved-azure-identity-patterns`, mirroring the existing `awsirsa` driver's
`autoApprovedARNPatterns`). A specific managed cluster's identity (its object ID and client ID) is
configured only on that cluster's own `Klusterlet`, in Story 2 below - never on the hub.

#### Story 2 - Managed cluster administrator joins a cluster using an Azure AD identity

I run my hub and managed clusters on AKS. Depending on which of the three Azure credential mechanisms
fits my environment, I select it explicitly with `--azure-credential` - all use the same `clusteradm
join` shape, differing only in the credential flag and what's supplied for the identity. This explicit
selection (rather than an automatic runtime fallback) is what lets the Helm chart expose four
distinct, independently-validated input shapes instead of one chart path whose effective behavior
depends on which environment variables happen to be set at deploy time:

**Managed Identity** - I'd rather use the identity already attached to my node than manage a
certificate lifecycle or set up a federated credential at all:

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential managed-identity-credential \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-client-id <client-id-of-user/system-assigned-identity> \
     --cluster-name managed-0
```

**Existing credential pipeline (environment variables)** - a client secret or certificate is already
provisioned by infrastructure I don't want to change. Neither the secret nor the certificate is ever
passed as a flag or stored in any OCM resource - only the identifying metadata below is; the driver
reads the credential material itself straight from the environment at runtime.

**Service Principal with Secret**

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-credential environment-credential-secret \
     --azure-tenant-id <tenant-id-of-service-principal> \
     --azure-client-id <client-id-of-service-principal> \
     --azure-client-secret <client-secret-of-service-principal> \
     --cluster-name managed-0
```

**Service Principal with Certificate**

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential environment-credential-certificate \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-tenant-id <tenant-id-of-service-principal> \
     --azure-client-id <client-id-of-service-principal> \
     --azure-client-cert-path <certificate-path-of-service-principal> \
     --azure-client-cert-password <certificate-password-of-service-principal> \
     --azure-client-send-cert-chain <true/false> \
     --cluster-name managed-0
```

**Workload Identity Federation** - I run under a policy that forbids storing long-lived credentials
on a cluster. My Azure AD identity's federated credential is already configured against my managed
cluster's Kubernetes ServiceAccount, so no client secret exists anywhere for this to leak:

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential workload-identity-credential \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-client-id <client-id-of-service-principal>  \
     --cluster-name managed-0
```

In every case, the agent authenticates to the hub without ever writing a certificate, token, or
secret to the managed cluster itself.

#### Story 3 - Hub administrator accepts a managed cluster's registration request

It must be possible for the hub administrator to accept the registration request using the `azure`
authentication strategy based on the presence of the `managed-cluster-azure-identity` annotation on
the `ManagedCluster`:

```
% clusteradm accept \
     --clusters managed-0
```

Equivalently, without `clusteradm`, the hub administrator can set `spec.hubAcceptsClient: true`
directly on the `ManagedCluster`, or enable the existing `ManagedClusterAutoApproval` feature gate
with a regex pattern matched against the Azure AD object ID recorded in that annotation.

#### Story 4 - Managed cluster administrator un-joins a cluster and cleans up hub resources

Run against the managed cluster's own context, this removes the klusterlet and its agents from the
managed cluster:

```
% clusteradm unjoin \
     --cluster-name managed-0
```

This does not by itself touch the hub. Deleting the `ManagedCluster` on the hub (or setting
`spec.hubAcceptsClient: false` on it) is what triggers the hub to remove the RBAC bindings created for
that cluster's identity - there is no separate `clusteradm` command for that side, since it's the same
hub-side cleanup path every other driver already goes through. There are no cloud-provider resources
(IAM roles, policies, federated credentials) for the hub to clean up, since none were created by the
hub controller in the first place - the Azure AD identity and its trust relationship continue to exist
independently of OCM.

#### Story 5 - Managed cluster permission isolation on hub

One managed cluster's Azure AD identity must not be able to access another managed cluster's
resources on the hub. This is provided by the existing per-cluster RBAC model unchanged: each
identity is bound only to the `ClusterRole`s scoped to its own cluster name/namespace, the same
isolation every other driver already relies on. This is a property of the existing RBAC model, not an
action anyone takes, so there is no corresponding `clusteradm` command either.

This isolation assumes each cluster is configured with a distinct Azure AD object ID. Nothing today
detects or rejects two different `ManagedCluster`s configured with the *same* object ID - see
[Risks and Mitigation](#risks-and-mitigation).

### Implementation Details/Notes/Constraints

#### Changes required on the managed cluster

A new package, `pkg/registration/register/azure_auth/`, provides a `RegisterDriver` implementation
that:
- Builds the kubeconfig's `exec` credential plugin to run a `get-azure-token` subcommand.
- Decorates the `ManagedCluster` it creates with a `managed-cluster-azure-identity` annotation holding
  the configured Azure AD object ID, so the hub can see which identity is claiming this cluster before
  any token exchange happens.

Because OCM can run the registration and work agents as one combined process (`Singleton` mode) or as
two separate processes (`Default` mode), and either can run on a separate management cluster
(`Hosted`/`SingletonHosted`), the process that ends up needing to run `get-azure-token` is not fixed.
The subcommand is registered on every binary that could end up in that position: `registration`,
`registration-operator` (which also serves as the `Singleton`-mode combined agent process), and
`work`.

#### Changes required on the hub

A `HubDriver` implementation binds the presented identity - as a Kubernetes `User`, keyed by its Azure
AD object (principal) ID - to the same three `ClusterRole`s a CSR-joined cluster is bound to, once
`hubAcceptsClient` is set to `true`:
- `open-cluster-management:managedcluster:<clusterName>` (`ClusterRoleBinding`) - status/identity
  permissions for the cluster's own `ManagedCluster` object.
- `open-cluster-management:managedcluster:<clusterName>:registration` (`RoleBinding` to the shared
  `open-cluster-management:managedcluster:registration` `ClusterRole`) - lease renewal, addon status.
- `open-cluster-management:managedcluster:<clusterName>:work` (`RoleBinding` to the shared
  `open-cluster-management:managedcluster:work` `ClusterRole`) - reading and updating `ManifestWork`.

No new `ClusterRole` is introduced; the identity is added as a subject on existing roles, the same
ones any other driver's accepted cluster is bound to.

Automatic approval reuses the existing `ManagedClusterAutoApproval` feature gate: a list of regex
patterns (`--auto-approved-azure-identity-patterns` on the hub, `azure.autoApprovedIdentityPatterns`
in the `ClusterManager` CR) is matched against the Azure AD object ID on the
`managed-cluster-azure-identity` annotation.

Unlike a design where the hub controller provisions cloud-provider identities dynamically per managed
cluster, no Azure resources are created, updated, or deleted by the hub controller at any point in
this flow. `Cleanup` only removes the Kubernetes RBAC bindings above.

#### Config surface

A new `authType: azure` option is added to the existing discriminated union already used by other
registration strategies, with a different sub-object shape on each side - there is no shared
"identity" schema between hub and spoke:

- **`Klusterlet.spec.registrationConfiguration.registrationDriver.azure`** (spoke), mirroring the
  flags Story 2 adds to `clusteradm join`. Two fields apply regardless of credential type:
  `azure.managedClusterAzureID` (required in every case - the Azure AD object/principal ID being
  claimed, used for hub-side auto-approval matching and RBAC binding) and `azure.tokenAudience`
  (optional in every case, defaulting to the well-known AKS AAD Server application when left unset -
  see [References](#references)). `azure.credential` selects which of the four `--azure-credential`
  values from Story 2 is in use, and each one has its own mandatory/optional fields - there is no
  single required-field set that applies uniformly across all four, since each corresponds to a
  distinct Helm chart input form that should only validate the fields it actually needs:
  - **`managed-identity-credential`**: `azure.clientID` is *optional* - set it to use a
    user-assigned identity, or omit it entirely to fall back to the node's system-assigned identity.
    No other fields apply.
  - **`environment-credential-secret`**: `azure.clientID`, `azure.tenantID`, and
    `azure.clientSecret` are all *required* - a secret-based service principal always has a client
    ID and tenant.
  - **`environment-credential-certificate`**: `azure.clientID`, `azure.tenantID`, and
    `azure.clientCertPath` are *required*; `azure.clientCertPassword` is *optional* (only needed if
    the certificate file is itself password-protected), and `azure.clientSendCertChain` is
    *optional*, defaulting to `false`.
  - **`workload-identity-credential`**: `azure.clientID` is *required* - unlike Managed Identity,
    there is no system-assigned equivalent for Workload Identity Federation (see [Managed cluster
    prerequisites](#managed-cluster-prerequisites)), so a specific identity must always be named.
    `azure.tenantID` and `azure.federatedTokenFile` are *optional*: both are normally auto-injected
    onto the pod by the Azure Workload Identity mutating webhook, and these fields exist only to
    override that.
- **`ClusterManager.spec.registrationConfiguration.registrationDriver.azure`** (hub):
  `azure.autoApprovedIdentityPatterns` (optional, a list of regex patterns matched against a joining
  cluster's Azure AD object ID for auto-approval), mirroring `awsirsa`'s `autoApprovedARNPatterns`.
  There is no per-managed-cluster identity field here - the hub's configuration applies uniformly to
  every cluster that joins with `authType: azure`; a specific cluster's identity lives only on that
  cluster's own `Klusterlet`.

#### Container image dependency

The Workload Identity Federation and environment-credential paths make one outbound network call:
the Azure AD token exchange itself (`https://login.microsoftonline.com`), a normal
public-CA-signed TLS connection. Any base image used to build the affected binaries needs a
standard root CA trust bundle for this to succeed - a certificate-based registration flow never
needs to validate a public CA, so a minimal base image built for that flow can omit one without
anything failing until this specific code path is exercised.

The Managed Identity path instead calls the Azure Instance Metadata Service at the link-local
address `169.254.169.254` over plain HTTP - no public CA involved, but only reachable from inside
Azure's own network (a VM, an AKS node, or equivalent). This path fails in any other environment
regardless of CA trust, and its failure mode is distinct from a CA trust problem - worth surfacing
as a separate, clearly-labeled error rather than a generic connection failure.

### Workflow Details

Actors:
1. cluster-admin on the managed cluster
2. Azure AD administrator (configures the identity and, for Workload Identity Federation, its
   federated credential - out of band, before any cluster join is attempted)
3. cluster-admin on the hub cluster
4. hub controller
5. agent on the managed cluster

#### Managed cluster prerequisites

An Azure AD identity - a user-assigned managed identity, a system-assigned managed identity, or an
app registration - known in advance to the managed cluster administrator by its client ID and
object ID. System-assigned managed identities only work with the Managed Identity credential path;
Workload Identity Federation requires a user-assigned managed identity or an app registration, since
a system-assigned identity has no separate resource to attach a federated credential to.

For Workload Identity Federation specifically, a federated identity credential on that identity with:
- **Issuer**: the managed cluster's own OIDC issuer URL.
- **Subject**: `system:serviceaccount:<agent-namespace>:<klusterlet-name>-registration-sa`, and a
  second federated credential with subject
  `system:serviceaccount:<agent-namespace>:<klusterlet-name>-work-sa` (in `Singleton`/`SingletonHosted`
  mode, registration and work share one ServiceAccount, so only the `-work-sa` subject is needed).
- **Audience**: `api://AzureADTokenExchange`.

This subject-matching is structurally the same idea as any federated-identity trust condition scoped
to a specific Kubernetes ServiceAccount - the identity can only be assumed by a token presented by
that exact ServiceAccount, in that exact namespace, nothing broader.

#### Hub cluster prerequisites

The hub *controller* needs nothing beyond what it already has: creating/updating
`ClusterRoleBinding`/`RoleBinding` objects is covered by its existing `ServiceAccount` permissions,
and it never calls out to Azure itself.

The hub **API server**, however, must independently be configured, out of band, to authenticate the
Azure AD access token the agent presents and map its `oid` (object ID) claim to a Kubernetes
username equal to that same value - either via AKS's native Azure AD integration, or by configuring
a self-managed apiserver to trust Azure AD as a generic OIDC issuer (`--oidc-issuer-url`,
`--oidc-client-id`, `--oidc-username-claim=oid`). `AzureAuthHubDriver.CreatePermissions` assumes
this trust already exists - it only manages the RBAC bindings on top of it, the same way the `csr`
driver assumes the apiserver already trusts the CA that signs its issued client certificates.

#### Cluster join initiated from the managed cluster

The join flow itself is the one the community already knows; azure-auth only changes what happens
around authentication:
- The klusterlet is deployed with `registrationConfiguration.registrationDriver.authType: azure`.
- The bootstrap call that creates `ManagedCluster` is unchanged - it still authenticates with the
  bootstrap kubeconfig's own shared, low-privilege ServiceAccount token, not yet an Azure AD token.
  Azure-auth's only addition at this step is setting the `managed-cluster-azure-identity` annotation to
  the configured Azure AD object ID.
- Once the hub accepts the request, the RBAC bindings created bind the Azure AD object ID as a `User`
  subject (see [Changes required on the hub](#changes-required-on-the-hub)) rather than relying on a
  certificate's Organization field.
- From that point on, the agent's kubeconfig `exec` plugin authenticates every call with a fresh Azure
  AD token fetched via `get-azure-token`, instead of a client certificate. Only the exec plugin's
  configuration (hub server address, CA data, command, and arguments) is ever persisted to the
  `hub-kubeconfig-secret` on the managed cluster - never a token, certificate, or other secret.

#### Un-join and cleanup

Un-join goes through the generic path the community already knows; azure-auth adds nothing to it.
The RBAC bindings removed there are the same ones azure-auth created on accept (see [Cluster join
initiated from the managed cluster](#cluster-join-initiated-from-the-managed-cluster)) - there's no
certificate to revoke, since none was ever issued, and nothing to clean up on the Azure side, since no
Azure resource was ever created by OCM in the first place.

### Risks and Mitigation

**Ownership and maintenance.** Azure-specific code requires Azure access and expertise to maintain,
which is why this proposal includes adding an `OWNERS` file scoped to the new package, naming a
dedicated maintainer for it distinct from the rest of the registration package.

**Silent failure mode across deployment shapes.** The pod label and ServiceAccount annotation that
enable Workload Identity Federation have to be correctly present on whichever manifest actually backs
the running agent pod, and that manifest differs across all four deployment shapes. Getting this wrong
for a shape other than the default doesn't produce a loud error - it silently falls through to a
different credential source, or fails much later, during actual registration rather than at deploy
time. Mitigated by explicit test coverage across all four shapes (see Test Plan).

**Shared, cross-binary credential state.** The `hub-kubeconfig-secret` written by one process (e.g. the
registration agent) may later be read by a different process (e.g. a separately-deployed work agent in
`Default` mode) that doesn't share the same binary. The exec plugin's command path in that persisted
config has to resolve correctly regardless of which binary reads it later - mitigated by every
relevant binary exposing `get-azure-token` at the same fixed path rather than assuming it can always
resolve its own executable path at write time.

**No duplicate-identity detection.** Nothing on the hub today rejects two different `ManagedCluster`s
configured with the same Azure AD object ID, on either the manual or automatic approval path. Should
that happen - administrator error, or a deliberately reused identity - that one Azure AD identity
would gain the union of both clusters' RBAC permissions, since both clusters' bindings target the
same Kubernetes `User` name, undermining the per-cluster isolation described in Story 5. Not
mitigated in this proposal; called out here as a known limitation. A follow-up could add an admission
check rejecting a `ManagedCluster` whose annotated object ID already has bindings for a different
cluster name.

### Test Plan

- Unit tests for the credential-chain fallback logic and the hub-side RBAC binding/approval logic, in
  isolation, without a real Azure identity.
- Integration tests (envtest) for the hub driver's `CreatePermissions`/`Cleanup`/`Accept` behavior.
- Verification against a real Azure identity, since the specific failure modes here are about real
  credential exchange and real RBAC - not something envtest or a fake client exercises. Before this is
  marked `implementable`, this should cover, at minimum:
  - Both Managed Identity and genuine Workload Identity Federation, against a real hub and managed
    cluster.
  - All four klusterlet deployment shapes (`Default`, `Singleton`, `Hosted`, `SingletonHosted`).
  - Both manual and automatic approval.
  - A real `ManifestWork` sent from the hub and confirmed applied on the managed cluster in each
    configuration above - status conditions alone ("Joined", "Available") are not sufficient
    verification, since a cluster can report healthy status while still missing the specific
    permission needed to receive any actual work.

### Graduation Criteria

New driver, opt-in by construction (`authType: azure` must be explicitly selected; every other
driver's behavior is unaffected). Proposed to ship directly as a supported driver once the Test Plan
above passes and an `OWNERS` file is in place, rather than progressing through a separate alpha/beta
gate.

### Upgrade / Downgrade Strategy

No impact on existing clusters. A cluster already registered via any other driver is unaffected by
this driver's addition; nothing here changes a shared code path those drivers rely on.

### Version Skew Strategy

Not applicable in a way that differs from adding any other driver - a hub or spoke predating this
change simply doesn't offer `authType: azure` as an option.

## Implementation History

## Drawbacks

Adds a new external dependency (Azure's identity SDK) and a new package that needs ongoing,
Azure-specific maintenance the existing maintainer group cannot provide directly - the ownership model
above exists specifically to address this.

## Alternatives

- CSR remains the preferred approach to managed cluster authentication with the hub, where usable.
- A design where the hub controller dynamically provisions Azure cloud resources (identities, role
  assignments) per managed cluster was considered and rejected in favor of relying entirely on
  pre-established Azure AD trust, keeping the hub controller's responsibility limited to
  Kubernetes-native RBAC (see Non-Goals).
- **Make registration auth pluggable out-of-tree instead of adding another in-tree driver.** Rather
  than growing the set of cloud-specific drivers shipped inside `ocm` itself, OCM core could expose
  the `RegisterDriver`/`HubDriver` interfaces as a stable extension point and ship only the `csr`
  driver as the built-in implementation. Every cloud- or environment-specific driver - this one,
  the existing AWS IAM driver, gRPC, and anything added later - would live and be maintained in its
  own separately-versioned repository, built and released independently by whoever actually needs
  and understands that specific cloud's identity model. This would directly address the maintenance
  concern raised in [ocm#1676](https://github.com/open-cluster-management-io/ocm/issues/1676): each
  driver's owner would carry its own release cadence, its own CI, and its own on-call for issues
  specific to that cloud, without needing OCM core maintainers to have expertise in every cloud a
  driver exists for.

  Not pursued as part of this proposal because it's a much larger, cross-cutting change than adding
  one driver: it would mean designing a real plugin/extension mechanism where none exists today (how
  an out-of-tree driver is discovered, loaded, and kept compatible with a core interface that evolves
  independently of it), and would imply migrating the already-shipped AWS IAM and gRPC drivers out of
  the core repository as well, not just deciding where this new one lands. Worth considering
  separately, as its own enhancement, rather than deciding it implicitly as a side effect of this one.

## References

- Azure Identity client library for Go (`azidentity`) - package documentation:
  https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azidentity
- Azure Workload Identity overview:
  https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- ocm#1676 (original proposal and community discussion):
  https://github.com/open-cluster-management-io/ocm/issues/1676
