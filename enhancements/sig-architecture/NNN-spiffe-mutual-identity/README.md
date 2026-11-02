# SPIFFE Mutual Identity for Hub, Klusterlet and Addons

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This enhancement adds a `spiffe` registration driver. In OCM today a managed cluster *earns* an
identity: it presents a bootstrap credential, requests a certificate, and the hub signs it with its
own CA. With the `spiffe` driver, every OCM component **already has** an identity, attested by
[SPIFFE/SPIRE](https://spiffe.io/docs/latest/spire-about/) from where and how it runs. Components
use those identities to **authenticate each other on every connection**:

- The klusterlet agent proves it is `spiffe://<td>/ocm/managedcluster/<c>/klusterlet`, and accepts
  the hub only if the hub proves it is `spiffe://<hub-td>/ocm/hub/grpc-server`.
- An addon agent proves it is `…/ocm/managedcluster/<c>/addon/<a>`, and accepts only the hub, or
  its own addon hub component, by SPIFFE ID.
- The hub maps each verified SPIFFE ID to the cluster or addon it belongs to. A component can act
  only as itself.

"Registration" reduces to recording which SPIFFE ID a `ManagedCluster` is bound to, and accepting
it. There is no bootstrap token, no CSR, no OCM certificate authority and no credential stored in
the Kubernetes API. Every credential is short-lived and exists only in process memory.

The design covers every deploy mode: Klusterlet `Default`, `Singleton`, `Hosted` and
`SingletonHosted`, and ClusterManager `Default` and `Hosted`. The connection between hub and spoke
needs no kube-apiserver configuration, so it works on self-managed Kubernetes and on managed
services such as AKS and EKS, and across fleets that span several SPIRE deployments.

## Motivation

OCM's registration drivers all follow the same pattern: bootstrap credential → certificate or
token issued by the hub → credential stored in a Secret → rotation before expiry. With the `grpc`
driver ([KEP 141](../141-grpc-based-registration/README.md)) that means:

| Where | Object | Contents | Lifetime |
|---|---|---|---|
| Hub | Secret `signer-secret` | Self-signed CA certificate **and private key**, used to sign agent client certificates, the gRPC serving certificate and the webhook serving certificates | 365 days |
| Hub | ConfigMap `ca-bundle-configmap` | CA certificates | Follows the CA |
| Hub | Secret `grpc-server-serving-cert`, webhook serving Secrets | TLS certificates and keys | 30 days |
| Hub | Secret `kube-system/bootstrap-token-ocmhub` (optional) | Bootstrap token | No expiry |
| Spoke | Bootstrap Secret | gRPC URL, CA certificate, bootstrap token | 1 hour, or none |
| Spoke | `hub-kubeconfig-secret` | Agent client certificate and key | Up to 30 days |
| Spoke | `<addon>-hub-kubeconfig` | Addon certificate and key, or a ServiceAccount token, plus the hub CA | Typically 1 year |
| Hosting cluster (Hosted Klusterlet) | `external-managed-kubeconfig` | Credential for the managed cluster's kube-apiserver, usually admin | Set by the user, often unlimited |
| Management cluster (Hosted ClusterManager) | `external-hub-kubeconfig`, `*-sa-kubeconfig` | Hub admin credential; ServiceAccount tokens | Set by the user; 360 days |

This causes five problems:

- **Identity is derived from a credential, not from the workload.** Whoever holds the bootstrap
  token, `signer-secret` or a copied `hub-kubeconfig-secret` *is* the cluster as far as the hub is
  concerned. Nothing checks where the caller is running.
- **Trust is one-way and pinned to a CA file.** Agents trust whatever the pinned OCM CA signed.
  They do not check *which* hub component they are talking to.
- **Slow revocation.** Agent certificates last up to 30 days, and the gRPC server does not check
  revocation.
- **Compliance.** Regulated environments increasingly forbid long-lived credentials and any
  credential material in Secrets or ConfigMaps, including public certificates.
- **Two identity systems.** Organisations that already run SPIRE must operate, audit and rotate a
  second, OCM-specific PKI just for OCM.

### Goals

**Identity requirements:**

| # | Requirement |
|---|---|
| I1 | Every OCM component that crosses a cluster boundary has a SPIFFE identity attested by SPIRE from the workload's own properties: cluster, namespace and ServiceAccount. |
| I2 | **Every connection is mutually authenticated.** Each side verifies the other's SPIFFE ID against an expected ID or a mapping, not merely against a CA. |
| I3 | An identity can act **only as itself.** A managed cluster cannot act as another cluster; an addon cannot act as its cluster's klusterlet or as another addon. This holds even if the cluster fully controls its own Kubernetes API. |
| I4 | A `ManagedCluster` is bound to exactly one SPIFFE ID. Rebinding needs an explicit administrator action. |

**Credential requirements:**

| # | Requirement |
|---|---|
| C1 | OCM generates, stores, mounts and uses **no certificate authority**. |
| C2 | **No OCM-created Secret or ConfigMap** contains a certificate (public or private), a private key or a token. |
| C3 | Every leaf credential OCM uses lasts **at most 1 hour** and is renewed automatically. A connection does not outlive the credential that opened it. |
| C4 | Every CA in the trust chain lasts **at most 24 hours**. Any longer-lived root key is held **outside the cluster**, in a KMS or HSM. |
| C5 | Credential material exists only in **process memory** or in pod-local **in-memory `emptyDir`** volumes. |
| C6 | Compliance is **testable**: an automated scan of all OCM Secrets and ConfigMaps finds no PEM blocks, JWTs or token Secrets. |

**Functional goals:**

- A managed cluster joins, is accepted, renews its lease and receives `ManifestWork` using only
  its SPIFFE identity.
- Addon agents have their own identity and reach the hub, or their addon's hub component, without
  stored credentials and without changes to addon images.
- **All deploy modes:** Klusterlet `Default`, `Singleton`, `Hosted` and `SingletonHosted`;
  ClusterManager `Default` and `Hosted`. The Hosted modes have one prerequisite, described in
  [Hosted modes](#hosted-modes).
- One hub can trust several SPIFFE trust domains.
- `clusteradm init` and `clusteradm join` support.
- Clusters move from `csr` or `grpc` to `spiffe` one at a time, without downtime.

### Non-Goals

- **Kubernetes' own control-plane credentials.** These are managed by Kubernetes or the cloud
  provider, not by OCM:
  - the apiserver, etcd and kubelet CAs;
  - the `kube-root-ca.crt` ConfigMap in every namespace;
  - kubelet-projected ServiceAccount tokens.
- **Installing or operating SPIRE.** This enhancement defines the SPIRE configuration it requires
  ([SPIRE profile](#spire-profile)) and documents it.
- **Changing the existing `csr`, `grpc` or `awsirsa` drivers.** They keep working as they do today.
- **Changes** to the CloudEvents event types or the SubjectAccessReview mapping.

## Proposal

### Overview

```text
 Managed cluster (or hosting cluster)               Hub (or management cluster)
┌──────────────────────────────────────┐          ┌───────────────────────────────────────────────┐
│ klusterlet agent                     │  mTLS    │ gRPC server, SPIFFE listener                  │
│  X.509-SVID                          │◄────────►│  X.509-SVID                                   │
│   …/ocm/managedcluster/c1/klusterlet │  both    │   …/ocm/hub/grpc-server                       │
│  accepts peer only if                │  sides   │  accepts peer only if its ID maps to a        │
│   ID == …/ocm/hub/grpc-server        │  verify  │   cluster or addon identity; pins the request │
│                                      │  IDs     │   to that cluster; SAR authorization          │
│ addon agent                          │          │                                               │
│  X.509-SVID …/c1/addon/my-addon      │◄────────►│ addon hub component (optional, direct)        │
│  accepts hub / addon hub by ID       │          │  X.509-SVID …/ocm/hub/addon/my-addon          │
│                                      │          │                                               │
│ SPIRE agent (Workload API via CSI)   │          │ SPIRE agent (Workload API via CSI)            │
└──────────────────────────────────────┘          └───────────────────────────────────────────────┘
        SPIRE: CA key in KMS/HSM, CA ≤ 24h, SVIDs ≤ 1h, trust bundles via Workload API + federation
```

The design has four parts:

1. **Identity.** SPIRE issues every OCM component an SVID based on the cluster, namespace and
   ServiceAccount it runs in. The SPIFFE ID encodes the cluster name and, for addons, the addon
   name.
2. **Mutual authentication.** All hub ↔ agent traffic goes through a dedicated SPIFFE listener on
   the gRPC server, using mTLS with X.509-SVIDs on both sides. Each side authorizes the peer's
   SPIFFE ID. Where a trust domain cannot be federated, a JWT-SVID can be used instead of the client
   certificate; the server is still verified by SVID.
3. **Registration as identity binding.** The agent announces itself over the authenticated
   connection. The hub creates or updates the `ManagedCluster` named in the SPIFFE ID, records the
   SPIFFE ID on it, and waits for acceptance. No credential is issued.
4. **Nothing stored.** No OCM CA, no serving-certificate Secrets, no bootstrap Secret and no
   credential Secret exists anywhere. What remains in Secrets and ConfigMaps is configuration only:
   URLs, SPIFFE IDs, socket paths, audiences and names.

### Registration: today vs. `spiffe`

| Step | `csr` / `grpc` today | `spiffe` |
|---|---|---|
| Prove who you are | Bootstrap token or kubeconfig, handed out of band | SVID attested by SPIRE from the workload's cluster, namespace and ServiceAccount |
| Prove who the hub is | Pinned CA certificate in the bootstrap Secret | Hub's SVID, checked against an expected SPIFFE ID |
| Get a steady-state credential | CSR → approved → signed by the OCM CA → stored in a Secret | None. The SVID is the credential and SPIRE rotates it in memory |
| Hub records the identity | Certificate subject; join is first-come | `ManagedCluster` is bound to one SPIFFE ID (I4) |
| Hub accepts | `hubAcceptsClient=true`, manual or auto-approval | Same; auto-approval can match SPIFFE ID patterns or a whole trust domain |
| Revoke | Wait up to 30 days, or remove RBAC | Delete the SPIRE entry: connections close within one SVID lifetime. Or remove RBAC: immediate |

### Credential model

| Credential | Issued by | Held in | Lifetime |
|---|---|---|---|
| SPIRE signing CA (X.509 and JWT keys) | SPIRE server, key in a KMS or HSM | KMS or HSM | ≤ 24h |
| gRPC server identity | SPIRE: X.509-SVID `spiffe://<hub-td>/ocm/hub/grpc-server` | gRPC server memory | ≤ 1h |
| Webhook serving identities | SPIRE: X.509-SVID `spiffe://<hub-td>/ocm/hub/<webhook>`, with DNS SANs | Webhook process memory | ≤ 1h |
| Klusterlet identity | SPIRE: X.509-SVID (or JWT-SVID) `…/ocm/managedcluster/<c>/klusterlet` | Agent memory | ≤ 1h |
| Addon identity | SPIRE: X.509-SVID (or JWT-SVID) `…/ocm/managedcluster/<c>/addon/<a>` | Agent or token-helper memory | ≤ 1h |
| Addon credential to the hub kube-apiserver (only for addons that need it) | Hub `TokenRequest`, through the gRPC server | Addon pod's in-memory `emptyDir` | 10 min – 1h |
| Hosted components → remote kube-apiserver | SPIRE: JWT-SVID, through an `exec` credential plugin | Process memory | ≤ 1h |
| Trust bundles | SPIRE Workload API (federated where trust domains differ) | Process memory | Follows the CA |
| Hub components → local hub kube-apiserver | Kubelet (projected ServiceAccount token) | Projected volume | ~1h |

### SPIFFE ID scheme

| Component | SPIFFE ID |
|---|---|
| Klusterlet agents (registration, work, or the singleton agent) | `spiffe://<td>/ocm/managedcluster/<c>/klusterlet` |
| Addon agent | `spiffe://<td>/ocm/managedcluster/<c>/addon/<a>` |
| Klusterlet operator on a hosting cluster (Hosted modes) | `spiffe://<td>/ocm/hosting/<h>/klusterlet-operator` |
| gRPC server | `spiffe://<hub-td>/ocm/hub/grpc-server` |
| Hub webhooks | `spiffe://<hub-td>/ocm/hub/{registration,work,addon}-webhook` |
| Cluster-manager operator and hub controllers | `spiffe://<hub-td>/ocm/hub/{cluster-manager,registration-controller,work-controller,placement-controller,addon-manager}` |
| Addon hub component | `spiffe://<hub-td>/ocm/hub/addon/<a>` |

`<c>`, `<h>` and `<a>` must be DNS-1123 labels. The `/ocm/managedcluster` prefix is configurable.

#### Mapping to OCM identities

The hub maps an authenticated SPIFFE ID to the user and group strings that OCM's existing
authorization already understands:

| SPIFFE ID | User | Groups |
|---|---|---|
| `…/ocm/managedcluster/<c>/klusterlet` | `system:open-cluster-management:<c>:klusterlet` | `system:open-cluster-management:<c>`, `system:open-cluster-management:managed-clusters`, `<bootstrapGroup>` |
| `…/ocm/managedcluster/<c>/addon/<a>` | `system:open-cluster-management:cluster:<c>:addon:<a>:agent:<a>-agent` | `system:open-cluster-management:cluster:<c>:addon:<a>`, `system:open-cluster-management:addon:<a>` |

Rules:
- Every other SPIFFE ID is rejected with `Unauthenticated`. A raw SPIFFE ID is never used as a
  username, because `ValidateClusterIdentity` only checks identities with the
  `system:open-cluster-management:` prefix and would let it through unpinned.
- Both user formats are ones `ValidateClusterIdentity` already parses, so every request is pinned
  to the cluster named in the SPIFFE ID before authorization runs (I3).
- The addon strings match addon-framework's `agent.DefaultUser` and `agent.DefaultGroups`, so
  existing addon RBAC applies unchanged.

The groups are the same ones CSR-issued certificates carry, so the `managedcluster` controller's
group-based RBAC applies unchanged, and so does the SubjectAccessReview authorizer.

### Identity isolation

Mapping a SPIFFE ID to a cluster name is only meaningful if nobody else can obtain that SPIFFE ID.
That must be enforced by SPIRE, not by anything a spoke administrator controls:

> **An SVID under `/ocm/managedcluster/<c>/` can be issued only to workloads on the cluster that
> runs `<c>`'s agents**: the managed cluster itself, or, in Hosted modes, its hosting cluster.

Supported topologies:

- **Shared trust domain, central SPIRE server.**
  - Each cluster's SPIRE agents attest directly to the central server with `k8s_psat`, one
    `clusters` entry per cluster.
  - Each cluster has a node alias selected by `k8s_psat:cluster:<name>`.
  - Every OCM entry for cluster `<c>` is parented to the alias of the cluster that runs its agents.
- **One trust domain per environment or per cluster**, joined by SPIFFE federation. The rule above
  applies inside each trust domain that serves several clusters.

Not supported, because each lets a spoke mint other clusters' IDs:
- a nested SPIRE server on a spoke that shares a trust domain with other clusters;
- SPIRE Controller Manager running on a spoke against a shared server.

In Hosted modes the hosting cluster can obtain the identity of every cluster it hosts. This is
deliberate: it already runs those agents, and today it holds their kubeconfigs. The difference is
that the SPIRE administrator grants this explicitly, entry by entry.

### SPIRE profile

A SPIRE deployment used with this enhancement must satisfy the following:

1. **No CA key in the cluster (C4).**
   - `KeyManager` is `azure_key_vault`, `aws_kms`, `gcp_kms`, or `memory`. `memory` loses keys on
     restart and forces a bundle roll. It is never `disk`.
   - Any `UpstreamAuthority` keeps its root key outside the cluster.
2. **Short lifetimes (C3, C4).**
   - `ca_ttl` ≤ `24h`.
   - `default_x509_svid_ttl` ≤ `1h`.
   - OCM entries set `-jwtSVIDTTL` ≤ `3600`; 1800 is recommended.
3. **No bundle in a ConfigMap (C2).**
   - No `k8sbundle` notifier.
   - SPIRE agents bootstrap trust with `trust_bundle_url` over HTTPS verified by the image's system
     roots. They do not use a ConfigMap-backed `trust_bundle_path`.
4. **Federation.** When the hub and spokes are in different trust domains, the SPIRE servers
   federate using the `https_spiffe` bundle endpoint profile, in both directions for mTLS, or
   hub → spoke only when spokes use JWT-SVIDs.
5. **Workload API access** through the [SPIFFE CSI driver](https://github.com/spiffe/spiffe-csi). It
   needs no `hostPath` and is compatible with Pod Security Admission `restricted`.
6. **Hosted modes only:** a [JWT issuer](#hosted-modes) (`jwt_issuer`) and the SPIRE OIDC
   discovery provider, reachable by the remote kube-apiserver.

### User Stories

#### Story 1: Hub administrator sets up the hub

The SPIRE administrator has deployed SPIRE according to the profile and created entries for the hub
components. The hub administrator runs:

```bash
clusteradm init --registration-drivers spiffe \
  --spiffe-trust-domain onprem.example.org \
  --spiffe-trust-domain cloud.example.org \
  --spiffe-auto-approve 'spiffe://onprem.example.org/ocm/managedcluster/prod-eu-*/klusterlet'
```

The cluster-manager operator:
- deploys the gRPC server with a SPIFFE listener, and the webhooks with SVID-based TLS;
- creates no CA, no serving-certificate Secrets and no bootstrap token;
- keeps the webhook `caBundle` fields in sync with the SPIRE bundle.

#### Story 2: Managed cluster administrator joins a cluster

The SPIRE administrator has created the cluster's node alias and OCM entries. The managed cluster
administrator runs:

```bash
clusteradm join --cluster-name prod-eu-1 --registration-auth spiffe \
  --hub-grpc-url grpc.hub.example.com:443 \
  --hub-spiffe-id spiffe://cloud.example.org/ocm/hub/grpc-server
```

There is nothing secret to pass, so there is no bootstrap Secret: the URL and the hub's SPIFFE ID
go in the `Klusterlet` spec. The agent:
1. checks that its own SVID is `…/ocm/managedcluster/prod-eu-1/klusterlet`;
2. connects and accepts the server only if it presents
   `spiffe://cloud.example.org/ocm/hub/grpc-server`;
3. announces itself. The hub creates `ManagedCluster/prod-eu-1` bound to the agent's SPIFFE ID.

#### Story 3: Hub administrator accepts the cluster

A hub administrator runs `clusteradm accept --clusters prod-eu-1`, after checking the bound SPIFFE
ID shown by `clusteradm get clusters`. Alternatively, auto-approval matches the SPIFFE ID. The
cluster starts syncing. No certificate is requested or issued at any point.

#### Story 4: Hosted klusterlet

Cluster `edge-7` is managed in `Hosted` mode from hosting cluster `mgmt-1`. Its kube-apiserver
trusts the SPIRE JWT issuer. The `edge-7` administrator grants the hosting operator's SPIFFE ID
access once, with a `ClusterRoleBinding`. The hosting administrator creates the `Klusterlet` with
`edge-7`'s apiserver URL. No kubeconfig is handed over, and the agents on `mgmt-1` reach both the
hub and `edge-7` using SVIDs.

#### Story 5: An addon agent and its hub component

The `my-addon` agent connects to its own hub component (for example a proxy or a metrics
receiver). Both use the [addon identity helper](#addon-identities): the agent accepts only
`spiffe://<hub-td>/ocm/hub/addon/my-addon`, and the hub component accepts only
`…/ocm/managedcluster/*/addon/my-addon` and learns the cluster name from the peer's SPIFFE ID.
Neither side ships or stores a certificate.

#### Story 6: Mixed on-prem and cloud fleet

On-prem clusters are in `onprem.example.org` and cloud clusters in `cloud.example.org`, each served
by its own SPIRE deployment. The two deployments federate. One AKS hub trusts both trust domains,
and neither environment can mint the other's identities. No Entra ID, ADFS or kube-apiserver
configuration is involved in OCM's trust.

#### Story 7: Compliance audit

An auditor runs `clusteradm check credentials` against the hub, each managed cluster and each
hosting cluster. It lists every Secret and ConfigMap in the OCM namespaces and reports none
containing a PEM block, a JWT or a token Secret type.

### Implementation Details/Notes/Constraints

#### Authentication on the gRPC server: `SPIFFEAuthenticator` (sdk-go)

A new authenticator in `sdk-go/pkg/server/grpc/authn` accepts two kinds of proof:

- **X.509-SVID (default).** The peer certificate from the mTLS handshake has already been verified
  against the X.509 bundles of the trusted trust domains, through go-spiffe
  `tlsconfig.MTLSServerConfig` with an authorizer that accepts only the configured trust domains and
  path prefix. The authenticator reads the SPIFFE ID from the URI SAN and the expiry from
  `NotAfter`.
- **JWT-SVID** (per trust domain, opt-in). The bearer token from the `authorization` metadata is
  verified with go-spiffe `jwtsvid.ParseAndValidate` against the configured audiences and the trust
  domain's JWT bundle. That bundle comes from the Workload API or from the SPIRE OIDC discovery
  provider (see [JWT bundles](#jwt-bundles)).

It then applies the [mapping table](#mapping-to-ocm-identities) and sets the user, the groups, and
two new context values: `ContextSPIFFEIDKey` and `ContextExpiryKey`.

**Connection lifetime (C3).** A TLS connection, and a `Subscribe` stream, stay open after the
certificate or token that opened them expires. The SPIFFE listener therefore:
- closes each stream with `Unauthenticated` when `ContextExpiryKey` passes, plus a small skew
  allowance;
- sets `MaxConnectionAge` below the SVID lifetime, so connections are re-handshaken with a current
  SVID.

The CloudEvents client already reconnects and resyncs. Revoked identities lose access within one
SVID lifetime.

#### The SPIFFE listener

The gRPC server gets a second listener, enabled when the `spiffe` driver is on:

- **Port.** Separate from the existing listener (default `8091`), so that `grpc` clients that pin
  the OCM CA keep working during migration. Once no other driver is enabled, the operator removes
  the old listener and the OCM CA (see [Upgrade](#upgrade--downgrade-strategy)).
- **TLS.** `tlsconfig.MTLSServerConfig(x509Source, bundleSet, authorizer)`, or
  `tlsconfig.TLSServerConfig(x509Source)` for trust domains that use JWT-SVIDs. Certificates come
  from `workloadapi.X509Source` and live only in memory.
- **Authentication and authorization.** `SPIFFEAuthenticator`, then the existing SAR authorizers.
- **Load balancers** in front of it must pass TLS through at layer 4.

sdk-go `GRPCServerOptions` gains `spiffe: {socket_path, trust_domains, path_prefix, ...}`.

#### JWT bundles

For trust domains configured with `clientAuthentication: JWTSVID`:

- **`WorkloadAPI`.** JWT bundles for the hub's own and federated trust domains come from the gRPC
  server's local SPIRE agent (`workloadapi.JWTSource`). No HTTP fetch is involved.
- **`OIDCDiscovery`.**
  - Fetches `<issuerURL>/.well-known/openid-configuration`, then `jwks_uri`, from the SPIRE OIDC
    discovery provider. Checks that `iss` equals `issuerURL`.
  - Refreshes keys periodically and when an unknown `kid` arrives, with rate limiting.
  - No CA bundle can be configured. The endpoint's TLS certificate is verified against the system
    roots in the image, or, when `endpointSPIFFEID` is set, as an X.509-SVID against the Workload
    API bundle.

JWT-SVIDs are for trust domains that can only be federated one way (hub → spoke). Spokes still
verify the server's SVID, so authentication remains mutual.

#### Agent side: the `spiffe` registration driver

A new `RegisterDriver` in `pkg/registration/register/spiffe`, selected by `authType: spiffe`:

- **Client.** sdk-go `GRPCConfig` gains a `spiffe` block:

  ```yaml
  url: grpc.hub.example.com:443
  spiffe:
    socketPath: unix:///run/spiffe.io/csi/agent.sock
    serverID: spiffe://cloud.example.org/ocm/hub/grpc-server
    clientAuthentication: X509SVID   # or JWTSVID
    audience: ocm-grpc               # JWTSVID only
  ```

  The client dials with `tlsconfig.MTLSClientConfig(x509Source, bundleSource,
  tlsconfig.AuthorizeID(serverID))`, or with `TLSClientConfig` plus a `PerRPCCredentials` backed
  by `workloadapi.JWTSource` in JWT mode. It has no `caData`, `token`, `tokenFile`,
  `clientCertFile` or `clientKeyFile`.
- **Configuration.** The klusterlet operator renders this from `Klusterlet.spec.registrationConfiguration.registrationDriver.spiffe`.
  There is no bootstrap Secret, and bootstrap and steady-state configuration are the same.
- **`Process`** sends no CSR. It reports registration complete once the cluster is accepted.
- **`IsHubKubeConfigValid`** checks that the rendered configuration matches the spec, and that a
  freshly fetched SVID has the ID `…/ocm/managedcluster/<clusterName>/klusterlet`. A missing or
  wrong SPIRE entry is reported on a Klusterlet condition, `SPIFFEIdentityAvailable`.
- **`ManagedClusterDecorator`** leaves the cluster unchanged. The server records the identity.
- **`Fork`** returns the addon `spiffe` driver.
- **Socket.** The klusterlet operator mounts the SPIFFE CSI volume into the agents.

#### Identity binding on the hub

When a SPIFFE-authenticated caller creates or updates a `ManagedCluster`, the gRPC cluster service:

- sets `open-cluster-management.io/spiffe-id` to the authenticated SPIFFE ID, overwriting any
  client-supplied value;
- **rejects** the request if the cluster is already bound to a different SPIFFE ID (I4). This
  covers, for example, an identical cluster name appearing in a second trust domain.

The registration webhook rejects changes to the annotation from anyone except the gRPC server and
cluster administrators with a dedicated `rebind` permission. `clusteradm` shows the binding in
`get clusters` and offers `clusteradm rebind`.

#### Hub registration driver

`SPIFFEHubDriver` in `pkg/registration/register/spiffe`:
- runs no CSR controllers and reads no CA files;
- **`Accept`** matches the bound SPIFFE ID against `autoApprovedIdentities`, which accepts exact
  IDs, `*` path-segment globs, or a whole trust domain;
- **`CreatePermissions`** and **`Cleanup`** are no-ops, because per-cluster RBAC is group-based.

#### Bootstrap permissions

The authenticator adds `bootstrapGroup` (default `open-cluster-management:spiffe-bootstrappers`) to
klusterlet identities. The cluster-manager operator binds that group to the existing
`open-cluster-management:bootstrap` `ClusterRole`. `ValidateClusterIdentity` pins every request to
the cluster in the SPIFFE ID, so a cluster that has not yet been accepted can only create or read
the `ManagedCluster` with its own name.

#### Hub TLS: webhooks

The registration, work and addon admission and conversion webhooks load their serving certificates
from `workloadapi.X509Source`, through controller-runtime's `webhook.Options.TLSOpts`
(`GetCertificate`). Each webhook's SPIRE entry carries a DNS SAN for its Service, or for its
external address in Hosted ClusterManager mode.

The kube-apiserver verifies webhooks against the `caBundle` field on their configuration objects.
The cluster-manager operator:
- reads the hub trust domain's X.509 bundle from its own Workload API;
- writes it into the `caBundle` of OCM's `ValidatingWebhookConfiguration`s,
  `MutatingWebhookConfiguration`s and CRD conversion webhooks, whenever the bundle changes;
- includes every certificate in the bundle, so SPIRE's pre-published next CA is trusted before it
  is used.

`caBundle` is a field on these objects, not a Secret or ConfigMap. It is the only place an
OCM-related certificate (public, at most 24 hours) appears in the Kubernetes API, because
Kubernetes requires it in order to call webhooks. This is the one link where only one side is
verified: the kube-apiserver is not a SPIFFE workload, and webhooks authorize requests through
SubjectAccessReview as they do today.

The certificate-rotation controller does not run. `signer-secret`, `ca-bundle-configmap`,
`grpc-server-serving-cert` and the webhook serving Secrets are never created.

#### Addon identities

Every addon agent has its own SPIFFE ID, `…/ocm/managedcluster/<c>/addon/<a>`, and is mapped to the
addon's existing OCM identity. Addons use it in up to three ways:

1. **Addon agent ↔ gRPC server.** The addon's `ManagedClusterAddOn` status, lease and events go
   through the gRPC server's existing addon services, authenticated by the addon's own SVID. The
   cluster's klusterlet SVID cannot act for the addon, and the addon cannot act for the klusterlet.
2. **Addon agent ↔ addon hub component.** A new `pkg/addon/spiffe` helper in addon-framework gives
   both sides a ready mTLS configuration:
   - `AgentTLSConfig(addonName)`: accepts only `spiffe://<hub-td>/ocm/hub/addon/<a>`;
   - `HubTLSConfig(addonName)`: accepts only `…/ocm/managedcluster/*/addon/<a>`, and
     `ClusterFromPeer(ctx)` returns the cluster name from the verified SPIFFE ID.

   This replaces addon-specific CAs and `customSigner` registrations for addons that run their own
   hub endpoints.
3. **Addon agent → hub kube-apiserver** (only for addons that watch hub resources directly). The
   addon `spiffe` kubeClient driver builds on the ServiceAccount model of
   [KEP 167](../167-token-based-addon-registration/README.md):
   - The addon manager creates the addon ServiceAccount `<c>-<a>-agent`, and a `Role` allowing
     `create` on `serviceaccounts/token` with `resourceNames: [<c>-<a>-agent]`, bound to the
     addon's group. Only the addon's own SVID can mint its token.
   - A token-helper sidecar (the `spiffe-addon-token` subcommand of the registration binary)
     calls the gRPC server's `TokenRequest` service with the addon SVID, verifying the server's
     SVID. It writes `token` and `ca.crt` to an `emptyDir` with `medium: Memory`, and renews at 80%
     of the token's lifetime.
   - `expirationSeconds` is clamped to [600, `maxAddOnTokenExpirationSeconds`] (default and
     maximum 3600).
   - `<a>-hub-kubeconfig` contains only the hub URL and the two file paths. client-go re-reads
     them, so the addon image is unchanged.

The addon manager injects the CSI volume, and the sidecar where needed, into template addons
(`AddOnTemplate`). addon-framework addons opt in through a helper in their manifest function.

#### Hosted modes

In Hosted modes the agents run on one cluster and manage another. Their path to the hub is
unchanged: the agent's SVID is `…/ocm/managedcluster/<c>/klusterlet`, issued on the hosting cluster
because the SPIRE entry is parented to the hosting cluster's node alias. What Hosted modes add is a
**second remote kube-apiserver**: the managed cluster's (Hosted Klusterlet) or the hub's (Hosted
ClusterManager). Today that is reached with a stored kubeconfig.

With `spiffe`, components reach the remote kube-apiserver with their own JWT-SVIDs:

- **Prerequisite.** The remote kube-apiserver trusts the SPIRE JWT issuer, through
  `AuthenticationConfiguration` (Kubernetes 1.30+) or the provider's OIDC identity-provider
  setting. The claim mapping sets the username to the SPIFFE ID (`claims.sub`), with a prefix such
  as `spiffe:`.
- **Client.** The operator builds the rest config in memory, or renders a kubeconfig containing
  only the URL and an `exec` credential plugin (`ocm-spiffe-credential --audience <aud>`). The
  plugin fetches a JWT-SVID from the Workload API and returns it as an `ExecCredential` with its
  expiry. No token is written to disk.
- **Verifying the remote apiserver.** The component verifies the apiserver's serving certificate
  by one of the following, set in the spec as `serverTrust`:
  - `SPIFFE`: the apiserver serves an X.509-SVID (self-managed control planes, with spiffe-helper)
    and is checked against its expected SPIFFE ID. This gives full mutual identity.
  - `SystemRoots`: the apiserver serves a publicly trusted certificate.
  - `CABundle`: the apiserver's own control-plane CA, in the CR spec. It is not in a Secret or
    ConfigMap and is Kubernetes-managed (a non-goal), but it is a public certificate in the API,
    so it is documented as an exception.

**Hosted and SingletonHosted Klusterlet:**

1. The managed cluster administrator binds `spiffe:spiffe://<td>/ocm/hosting/<h>/klusterlet-operator`
   to a `ClusterRole` that lets the operator install OCM's CRDs, namespace and agent RBAC. This
   one-time binding replaces handing over an admin kubeconfig, and it can be revoked at any time.
2. The klusterlet operator on hosting cluster `<h>` uses its own SVID to install those resources on
   the managed cluster. The agent RBAC it installs is bound to
   `spiffe:spiffe://<td>/ocm/managedcluster/<c>/klusterlet`.
3. The agents on the hosting cluster reach the managed cluster with their JWT-SVIDs, and the hub
   with mTLS as in every other mode.
4. `external-managed-kubeconfig` is not used, and the operator's existing secret-based hosted path
   does not run.

**Hosted ClusterManager:**

1. The hub administrator binds `spiffe:spiffe://<hub-td>/ocm/hub/cluster-manager` to
   `cluster-admin` on the hub, or a narrower role.
2. The cluster-manager operator on the management cluster deploys the hub controllers there, each
   with its own SPIFFE ID (`…/ocm/hub/registration-controller` and so on). It installs hub RBAC for
   those IDs, replacing the `*-sa-kubeconfig` Secrets and their 360-day tokens.
3. The gRPC server and webhooks run on the management cluster with SVIDs, as in `Default` mode.
   The hub kube-apiserver reaches the webhooks at their external addresses and verifies them using
   the synced `caBundle`.
4. `external-hub-kubeconfig` and `*-sa-kubeconfig` are not used.

Clusters whose kube-apiserver cannot trust an external JWT issuer can still use `spiffe` in
`Default` or `Singleton` mode. They cannot be the managed cluster in Hosted Klusterlet mode, or the
hub in Hosted ClusterManager mode, without falling back to today's kubeconfig Secrets, which do not
meet C2.

#### API changes

**ClusterManager.** `RegistrationDriverHub.AuthType` gains `spiffe`, with a matching member:

```go
type RegistrationDriverHub struct {
    // +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
    AuthType string `json:"authType,omitempty"`
    // ... existing members ...

    // +optional
    SPIFFE *SPIFFEHubConfig `json:"spiffe,omitempty"`
}

type SPIFFEHubConfig struct {
    // +kubebuilder:validation:MinItems=1
    TrustDomains []SPIFFETrustDomain `json:"trustDomains"`

    // Exact SPIFFE IDs, IDs with '*' path segments, or "spiffe://<td>" to accept a whole
    // trust domain. Matched against the bound SPIFFE ID.
    // +optional
    AutoApprovedIdentities []string `json:"autoApprovedIdentities,omitempty"`

    // +kubebuilder:default="/ocm/managedcluster"
    PathPrefix string `json:"pathPrefix,omitempty"`

    // +kubebuilder:default="open-cluster-management:spiffe-bootstrappers"
    BootstrapGroup string `json:"bootstrapGroup,omitempty"`

    // +kubebuilder:validation:Minimum=600
    // +kubebuilder:validation:Maximum=3600
    // +kubebuilder:default=3600
    MaxAddOnTokenExpirationSeconds int64 `json:"maxAddOnTokenExpirationSeconds,omitempty"`
}

// +kubebuilder:validation:XValidation:rule="self.clientAuthentication != 'JWTSVID' || has(self.jwt)"
type SPIFFETrustDomain struct {
    Name string `json:"name"`

    // +kubebuilder:validation:Enum=X509SVID;JWTSVID
    // +kubebuilder:default=X509SVID
    ClientAuthentication string `json:"clientAuthentication,omitempty"`

    // +optional
    JWT *SPIFFEJWTConfig `json:"jwt,omitempty"`
}

// +kubebuilder:validation:XValidation:rule="self.bundleSource != 'OIDCDiscovery' || has(self.oidcDiscovery)"
type SPIFFEJWTConfig struct {
    // +kubebuilder:default={"ocm-grpc"}
    Audiences []string `json:"audiences,omitempty"`
    // +kubebuilder:validation:Enum=WorkloadAPI;OIDCDiscovery
    BundleSource  string               `json:"bundleSource"`
    OIDCDiscovery *SPIFFEOIDCDiscovery `json:"oidcDiscovery,omitempty"`
}

type SPIFFEOIDCDiscovery struct {
    // Must equal the SPIRE server's jwt_issuer and the discovery document's issuer.
    IssuerURL string `json:"issuerURL"`
    // When set, the endpoint's TLS certificate is verified as an X.509-SVID with this ID.
    // Otherwise it is verified against system roots. There is deliberately no CA bundle field.
    // +optional
    EndpointSPIFFEID string `json:"endpointSPIFFEID,omitempty"`
}
```

For Hosted ClusterManager, `HostedClusterManagerConfiguration` gains:

```go
// When set, hub components reach the hub kube-apiserver with SPIFFE JWT-SVIDs
// instead of external-hub-kubeconfig.
// +optional
HubAPIServer *SPIFFERemoteAPIServer `json:"hubAPIServer,omitempty"`
```

**Klusterlet.** `RegistrationDriver.AuthType` gains `spiffe`:

```go
// +kubebuilder:validation:XValidation:rule="self.authType != 'spiffe' || has(self.spiffe)"
type RegistrationDriver struct {
    // +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
    AuthType string `json:"authType,omitempty"`
    // ... existing members ...

    // +optional
    SPIFFE *SPIFFEAgentConfig `json:"spiffe,omitempty"`
}

type SPIFFEAgentConfig struct {
    // gRPC server SPIFFE listener address (host:port). Replaces the bootstrap Secret.
    ServerURL string `json:"serverURL"`
    // SPIFFE ID the gRPC server must present. Replaces a pinned CA.
    ServerID string `json:"serverID"`
    // +kubebuilder:validation:Enum=X509SVID;JWTSVID
    // +kubebuilder:default=X509SVID
    ClientAuthentication string `json:"clientAuthentication,omitempty"`
    // +kubebuilder:default=ocm-grpc
    Audience string `json:"audience,omitempty"`
    // Socket file inside the SPIFFE CSI volume.
    // +optional
    SocketPath string `json:"socketPath,omitempty"`
    // +kubebuilder:default="/ocm/managedcluster"
    PathPrefix string `json:"pathPrefix,omitempty"`
}
```

For Hosted and SingletonHosted, `Klusterlet.spec` gains:

```go
// When set, the operator and agents reach the managed cluster's kube-apiserver with
// SPIFFE JWT-SVIDs instead of external-managed-kubeconfig.
// +optional
ManagedClusterAPIServer *SPIFFERemoteAPIServer `json:"managedClusterAPIServer,omitempty"`
```

Shared type:

```go
// +kubebuilder:validation:XValidation:rule="self.serverTrust != 'SPIFFE' || has(self.serverID)"
// +kubebuilder:validation:XValidation:rule="self.serverTrust != 'CABundle' || has(self.caBundle)"
type SPIFFERemoteAPIServer struct {
    URL string `json:"url"`
    // Audience of the JWT-SVIDs, as configured in the remote AuthenticationConfiguration.
    Audience string `json:"audience"`
    // +kubebuilder:validation:Enum=SPIFFE;SystemRoots;CABundle
    ServerTrust string `json:"serverTrust"`
    // +optional
    ServerID string `json:"serverID,omitempty"`
    // The remote apiserver's own control-plane CA. Kubernetes-managed; see "Hosted modes".
    // +optional
    CABundle []byte `json:"caBundle,omitempty"`
}
```

CEL rules:
- Klusterlet `Hosted`/`SingletonHosted` with `authType: spiffe` requires `managedClusterAPIServer`.
- ClusterManager `Hosted` with a `spiffe` driver requires `hubAPIServer`.

Further changes:
- `AddOnRegistrationDriver.AuthType` gains `spiffe`. It is required to be `spiffe` when the cluster
  uses `authType: spiffe`.
- `ManagedClusterAddOn` `KubeClientConfig.Driver` gains `spiffe`.

All types land in `open-cluster-management-io/api` first.

#### SPIRE entries

Example registration entries, created by the trust-domain administrator:

```bash
# Hub components (hub trust domain)
spire-server entry create -spiffeID spiffe://cloud.example.org/ocm/hub/grpc-server \
  -parentID <hub-node-alias> \
  -selector k8s:ns:open-cluster-management-hub -selector k8s:sa:grpc-server-sa
spire-server entry create -spiffeID spiffe://cloud.example.org/ocm/hub/registration-webhook \
  -parentID <hub-node-alias> \
  -selector k8s:ns:open-cluster-management-hub -selector k8s:sa:registration-webhook-sa \
  -dns cluster-manager-registration-webhook.open-cluster-management-hub.svc
# work-webhook, addon-webhook, cluster-manager and the hub controllers follow the same pattern

# Managed cluster prod-eu-1 (Default or Singleton mode)
spire-server entry create -node \
  -spiffeID spiffe://onprem.example.org/spire/agent/k8s_psat/prod-eu-1 \
  -selector k8s_psat:cluster:prod-eu-1

for sa in klusterlet-registration-sa klusterlet-work-sa; do
  spire-server entry create \
    -spiffeID spiffe://onprem.example.org/ocm/managedcluster/prod-eu-1/klusterlet \
    -parentID spiffe://onprem.example.org/spire/agent/k8s_psat/prod-eu-1 \
    -selector k8s:ns:open-cluster-management-agent -selector k8s:sa:$sa
done

spire-server entry create \
  -spiffeID spiffe://onprem.example.org/ocm/managedcluster/prod-eu-1/addon/my-addon \
  -parentID spiffe://onprem.example.org/spire/agent/k8s_psat/prod-eu-1 \
  -selector k8s:ns:open-cluster-management-agent-addon -selector k8s:sa:my-addon-agent-sa

# Managed cluster edge-7, hosted on mgmt-1: parented to the *hosting* cluster
spire-server entry create \
  -spiffeID spiffe://onprem.example.org/ocm/hosting/mgmt-1/klusterlet-operator \
  -parentID spiffe://onprem.example.org/spire/agent/k8s_psat/mgmt-1 \
  -selector k8s:ns:open-cluster-management -selector k8s:sa:klusterlet
spire-server entry create -jwtSVIDTTL 1800 \
  -spiffeID spiffe://onprem.example.org/ocm/managedcluster/edge-7/klusterlet \
  -parentID spiffe://onprem.example.org/spire/agent/k8s_psat/mgmt-1 \
  -selector k8s:ns:klusterlet-edge-7 -selector k8s:sa:klusterlet-agent-sa
```

#### Registration flow

1. The klusterlet operator renders the agents from the `Klusterlet` spec, with the SPIFFE CSI
   volume.
2. The agent fetches its X.509-SVID and connects to the SPIFFE listener. Both sides verify each
   other's SVID; the agent requires the configured server ID.
3. The gRPC server:
   - maps the peer to `system:open-cluster-management:prod-eu-1:klusterlet`;
   - pins the request to `prod-eu-1`;
   - authorizes the `ManagedCluster` create through the bootstrap group;
   - binds the cluster to the SPIFFE ID, or rejects the request if it is bound to another.
4. The cluster is accepted, manually or by auto-approval, and the `managedcluster` controller binds
   the cluster group.
5. Lease, status and `ManifestWork` sync run over the same connection. Connections are
   re-established with a current SVID before each one expires.

**Revocation:**
- Deleting the cluster's SPIRE entries stops new SVIDs. Existing connections close when their SVID
  expires (at most one hour).
- Setting `hubAcceptsClient=false` removes the cluster's RBAC immediately.
- Removing the binding (`clusteradm rebind --clear`) blocks the old SPIFFE ID immediately.

### Risks and Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| SPIRE topology lets one spoke mint another cluster's IDs | High: cross-cluster impersonation | Documented isolation property and supported topologies; per-trust-domain authorization; identity binding (I4) catches a second trust domain claiming an existing name |
| Authenticator lets an unmapped identity through | High: bypasses `ValidateClusterIdentity` | Reject everything outside the mapping table; tests assert this |
| SPIRE outage | New connections fail within one hour; webhooks fail within one CA lifetime | Highly available SPIRE servers; KMS-backed keys survive restarts; documented outage budget; the existing lease grace period |
| Webhook `caBundle` falls behind a CA rotation | Webhook calls fail | SPIRE pre-publishes the next CA; the operator syncs on every bundle change and includes all certificates; alert when the bundle is older than half the CA lifetime |
| Reconnect load from one-hour SVIDs | Connection churn at fleet scale | Jittered reconnects; scale test before Beta |
| Hosting cluster compromised | It can act as every cluster it hosts | Same as today with stored kubeconfigs, but explicit and per-entry; documented |
| Remote apiserver can't trust an external JWT issuer | Hosted modes unavailable for that cluster | `Default`/`Singleton` modes; documented |
| Webhook `caBundle` holds a public certificate | Visible in the API, though not a Secret or ConfigMap | Required by Kubernetes; at most 24 hours; documented |

## Design Details

### Open Questions

1. **Hub apiserver CA for addons.** Addons that watch hub resources directly still need the hub
   kube-apiserver's CA. It is Kubernetes-managed and long-lived, but here it is held only in pod
   memory. Should such addons instead reach hub resources only through the gRPC server?
2. **Remote apiserver trust in Hosted modes.** Is `CABundle` in the CR spec acceptable, or should
   Hosted modes require `SPIFFE` or `SystemRoots`?
3. **Entry management.** Should a hub controller with SPIRE server API access create cluster and
   addon entries from `ManagedCluster` and `ManagedClusterAddOn` objects, as a separate
   enhancement? This would make join a single step but would give the hub write access to SPIRE.
4. **Trust-domain auto-approval.** Accepting a whole trust domain treats "the SPIRE administrator
   created an entry" as the hub's opt-in. Is that an acceptable default for single-tenant fleets?

### Test Plan

- **Unit (sdk-go):**
  - the authenticator with X.509-SVIDs and JWT-SVIDs: valid, expired, wrong trust domain,
    unmapped path, wrong audience, wrong issuer and unknown `kid`;
  - both JWT bundle sources;
  - closing streams at expiry and `MaxConnectionAge`;
  - client `AuthorizeID` rejecting a server with a different SPIFFE ID.
- **Unit (ocm):**
  - the `spiffe` register and hub drivers;
  - identity binding and rebinding;
  - TokenRequest restrictions and clamping;
  - the operator rendering no CA, serving-certificate, bootstrap or kubeconfig Secrets in each
    deploy mode;
  - webhook `caBundle` sync;
  - the `ocm-spiffe-credential` exec plugin;
  - the CEL rules;
  - the addon-framework `pkg/addon/spiffe` helper.
- **Integration (envtest):**
  - Run the gRPC server with a fake Workload API that issues X.509- and JWT-SVIDs from a CA with a
    one-hour lifetime, and an in-process OIDC issuer.
  - Cover join, binding, acceptance, sync, reconnect at expiry, CA rotation during operation, the
    addon token exchange, and impersonation attempts that must fail: another cluster's name,
    an addon acting as its klusterlet, and a second trust domain claiming a bound name.
- **E2E (kind):**
  - Two federated SPIRE trust domains, `KeyManager "memory"`, and `ca_ttl: 1h` to exercise
    rotation.
  - Klusterlet `Default`, `Singleton` and `Hosted`, with a template addon and an addon using the
    direct mTLS helper.
  - A managed kind cluster whose apiserver uses `AuthenticationConfiguration` to trust the SPIRE
    issuer, for the Hosted case; the same for Hosted ClusterManager.
  - **The C6 scan**, which fails on:
    - any PEM block or JWT-shaped value in a Secret or ConfigMap in the OCM hub, agent, addon and
      hosting namespaces;
    - any `bootstrap.kubernetes.io/token` or `kubernetes.io/service-account-token` Secret.

    `kube-root-ca.crt` is excluded because Kubernetes manages it.

### Graduation Criteria

**Alpha:**
- the authenticator, SPIFFE listener and client in sdk-go;
- the `spiffe` register and hub drivers, behind a feature gate;
- identity binding;
- Klusterlet `Default` and `Singleton`;
- integration tests;
- the C6 scan on the hub ↔ agent path.

**Beta:**
- webhooks on SVIDs with `caBundle` sync;
- addon identities: the gRPC addon path, the mTLS helper and the `spiffe` kubeClient driver;
- Hosted Klusterlet and Hosted ClusterManager;
- `clusteradm` support;
- full e2e, including the C6 scan across all OCM namespaces;
- a reconnect scale test at 1,000 clusters.

**GA:**
- two production adopters, including one using a Hosted mode;
- a security review including a SPIFFE/SPIRE maintainer;
- the API unchanged for two releases.

### Upgrade / Downgrade Strategy

`spiffe` is a separate driver, so the hub can enable it alongside `csr` or `grpc` and clusters move
one at a time:

1. Add `spiffe` to the ClusterManager's `registrationDrivers`. The operator opens the SPIFFE
   listener next to the existing one. Nothing else changes.
2. Move each cluster: create its SPIRE entries, then set the Klusterlet to `authType: spiffe`. The
   agent reconnects on the SPIFFE listener and the hub binds the existing `ManagedCluster` to its
   SPIFFE ID. The cluster group is the same, so no RBAC changes are needed and the cluster stays
   accepted.
3. Move each addon to `spiffe`.
4. Remove the other drivers from `registrationDrivers`. The operator refuses while any accepted
   cluster or addon still uses them, and names those on a condition. Once they are removed, the
   operator:
   - closes the old listener;
   - stops the CSR signer and the certificate-rotation controller;
   - deletes `signer-secret`, `ca-bundle-configmap`, the serving-certificate Secrets and the
     bootstrap token;
   - switches the webhooks to SVIDs.

   Only from this point does the hub meet C1–C6.

Hosted users move a hosted cluster by adding `managedClusterAPIServer` (or `hubAPIServer`) together
with `authType: spiffe`. The operator stops using the kubeconfig Secret and deletes the copies it
created. The user deletes the Secret they supplied.

**Downgrade:** re-add `csr` or `grpc`. The operator recreates the CA and serving certificates, after
which clusters can move back. Bindings remain and are ignored by other drivers.

### Version Skew Strategy

- **A new agent with `authType: spiffe` against a hub without the SPIFFE listener** fails to
  connect. It reports this on the `SPIFFEIdentityAvailable` or `HubConnectionDegraded` condition and
  keeps retrying.
- **Old agents** keep using their existing driver and listener. Step 4 of the upgrade is gated on
  none remaining.
- **The addon `spiffe` driver** requires a gRPC server and an addon manager that support it.
  Otherwise the addon stays unregistered with a condition naming the unsupported driver, and the
  cluster's own registration is unaffected.

## Implementation History

- 2026-10-31: Initial proposal, as SPIFFE credentials for the `grpc` driver.
- 2026-11-02: Reworked as a separate `spiffe` driver built on mutual identity; added Hosted modes and
  direct addon ↔ addon hub authentication.

## Drawbacks

- **SPIRE becomes a hard runtime dependency.** With one-hour credentials and a CA of at most 24
  hours, a prolonged SPIRE outage stops the fleet. Today's 30-day certificates tolerate outages for
  longer.
- **The security of cluster identities depends on SPIRE's topology and configuration**, which OCM
  can only partly verify.
- **Hosted modes need a configurable remote kube-apiserver**, which rules out some managed
  services as the managed cluster or hub in those modes.
- **Another registration driver** and a second gRPC listener to maintain alongside the existing
  ones.

## Alternatives

- **SPIFFE as a credential option inside the `grpc` driver.** This was the first version of this
  proposal. It keeps the bootstrap-and-CSR shape, the shared listener makes the OCM CA hard to
  retire, and it doesn't model identity binding or addon-to-addon trust.
- **SPIFFE for agent authentication only, keeping OCM's CA for server TLS.** Simpler, but agents
  still trust a CA rather than a specific hub identity, and a one-year CA key stays in a Secret. It
  does not meet I2, C1, C2 or C4.
- **Authenticate SPIFFE at the hub kube-apiserver** with `AuthenticationConfiguration` and a
  kube-apiserver-based driver. This only works where the hub apiserver can be configured, puts
  agents back in front of the apiserver (against KEP 141's goal), and still leaves agents trusting a
  CA rather than a SPIFFE ID. Managed hubs would need a cloud token exchange per cluster.
- **TLS terminated at a cloud load balancer with a managed certificate.** Agents would trust web
  PKI and a hostname rather than a SPIFFE ID, client authentication would be lost at the load
  balancer, and it doesn't cover on-prem hubs.
- **cert-manager with a KMS-backed issuer.** Certificates and keys are still stored in Secrets,
  and identity still comes from holding a credential.
- **A small proxy on the managed cluster for Hosted Klusterlet**, so the remote apiserver needs no
  configuration. It defeats the purpose of Hosted mode, which is to run nothing on the managed
  cluster.

## References

- [KEP 141: gRPC based registration](../141-grpc-based-registration/README.md)
- [KEP 167: token-based addon registration](../167-token-based-addon-registration/README.md)
- [SPIFFE concepts](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/)
- [SPIRE server configuration](https://spiffe.io/docs/latest/deploying/spire_server/) (`ca_ttl`, `default_x509_svid_ttl`, `jwt_issuer`, KeyManager plugins)
- [SPIRE agent configuration](https://spiffe.io/docs/latest/deploying/spire_agent/) (`trust_bundle_url`)
- [SPIFFE federation](https://spiffe.io/docs/latest/architecture/federation/readme/) (`https_spiffe` bundle endpoint profile)
- [SPIRE OIDC discovery provider](https://github.com/spiffe/spire/tree/main/support/oidc-discovery-provider)
- [go-spiffe v2](https://github.com/spiffe/go-spiffe) (`workloadapi.X509Source`, `workloadapi.JWTSource`, `tlsconfig`, `jwtsvid`)
- [SPIFFE CSI driver](https://github.com/spiffe/spiffe-csi)
- [Kubernetes structured authentication configuration](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#using-authentication-configuration)
- `open-cluster-management.io/sdk-go`: `pkg/server/grpc/authn`, `pkg/server/grpc/options.go`, `pkg/cloudevents/server/grpc/authz/kube/sar.go`
- `open-cluster-management.io/ocm`: `pkg/server/grpc`, `pkg/registration/register`, `pkg/operator/operators/clustermanager/controllers/certrotationcontroller`, `pkg/operator/helpers/sa_syncer.go`
