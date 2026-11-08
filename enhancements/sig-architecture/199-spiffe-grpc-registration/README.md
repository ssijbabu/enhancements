# SPIFFE gRPC registration

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal adds a new registration driver, `spiffe`, in which every OCM agent identity is a
short-lived [SPIFFE](https://spiffe.io/) SVID issued by [SPIRE](https://spiffe.io/docs/latest/spire-about/).
Klusterlet agents and add-on agents connect to the hub gRPC server over mutual TLS with their own
X.509-SVIDs, and the hub gRPC server authenticates with its own X.509-SVID. Registration, renewal
and revocation all work without a bootstrap token, without a CertificateSigningRequest, without a
hub signing key, and without any Secret or ConfigMap that holds a private key, public key or
certificate. The only identity service OCM talks to is the local SPIRE agent, through the SPIFFE
Workload API.

The driver works in all four Klusterlet install modes (`Default`, `Singleton`, `Hosted` and
`SingletonHosted`) and applies the same model to add-ons.

The driver is new. It reuses the CloudEvents-over-gRPC transport, the per-resource gRPC services
and the SubjectAccessReview authorizer from [gRPC based registration](../141-grpc-based-registration/README.md),
but replaces its entire identity layer.

## Motivation

The existing registration drivers (`csr`, `grpc`, `awsirsa`) and add-on registration drivers
(`csr`, `token`) all leave long-lived or reusable credentials in OCM's hands:

- A bootstrap kubeconfig or bootstrap token must be distributed to each managed cluster and is
  valid until someone removes it.
- The `csr` and `grpc` drivers have the hub sign client certificates, so the hub holds a signing
  key (`signer-secret`) and the hub gRPC server holds a serving certificate in a Secret.
- The agent's private key and certificate are written to the `hub-kubeconfig-secret`, and each
  add-on's credentials to `<addon>-hub-kubeconfig` or `<addon>-<signer>-client-cert` Secrets.
  Anyone who can read those Secrets can act as the cluster or the add-on.
- Renewal is OCM's job (CSR rotation), and revoking a cluster or add-on does not invalidate a
  certificate that was already issued.

Many platforms already run SPIRE as their workload identity system. In those environments the
platform wants a single place that issues and rotates identities, short credential lifetimes, and
no key material at rest. OCM should be able to plug into that model instead of running its own
certificate authority.

### Goals

- Add a `spiffe` registration driver to the hub (`ClusterManager`) and the agent (`Klusterlet`).
- Use only SPIRE-issued SVIDs for every Klusterlet agent, every add-on agent and the hub gRPC
  server, for the whole lifecycle: register, renew, revoke and unjoin.
- Support all four Klusterlet install modes: `Default`, `Singleton`, `Hosted` and
  `SingletonHosted`.
- Support add-ons with the same model: each add-on agent registers, renews and is revoked using
  only its own SVID.
- Agents hold only short-lived SVIDs, in memory. Nothing OCM creates stores key material.
- OCM does not create Secrets or ConfigMaps for private keys, public keys or certificates, and does
  not sign anything.
- Apart from the local SPIRE agent's Workload API, OCM contacts no other identity or key service
  (no KMS, cloud identity provider, OIDC endpoint or SPIRE server API).
- Keep the hub-side user experience unchanged: ManagedCluster acceptance, add-on enablement, RBAC,
  ManifestWork and add-on APIs work as today.
- Allow existing clusters to move to the new driver one at a time.

### Non-Goals

- Installing, configuring or operating SPIRE. SPIRE topology, node attestation and registration
  entries are the platform's responsibility; this proposal only defines what OCM expects of them.
- Changing the existing `csr`, `grpc` or `awsirsa` drivers, or the `csr` and `token` add-on
  drivers.
- Giving agents or add-ons a kubeconfig for the hub kube-apiserver. In this mode they reach the
  hub only through the gRPC server.
- `customSigner` add-on registrations. They exist to have the hub sign client certificates; in
  this mode an add-on that needs a client certificate uses its SVID instead.
- JWT-SVID authentication (see [Alternatives](#alternatives)).

## Proposal

![Components and trust in SPIFFE gRPC registration](spiffe-grpc-registration-overview.svg)

- **SPIRE** issues every identity. A SPIRE agent runs on each node of the hub, the managed
  clusters and, in hosted modes, the management clusters, and exposes the Workload API through
  the SPIFFE CSI driver.
- **Each OCM workload gets its own SVID.** The hub gRPC server, every Klusterlet agent and every
  add-on agent fetch their own X.509-SVID and trust bundle from the Workload API of the cluster
  they run on. SPIRE pushes a new SVID before the current one expires.
- **The hub gRPC server** serves TLS with its SVID, requires a client SVID from the configured
  trust domain, maps the client's SPIFFE ID to an OCM user and groups, and authorizes every
  CloudEvent with SubjectAccessReview, as today.
- **Agents** verify that the server presents the exact hub SPIFFE ID, so a load balancer or a
  certificate from the same trust domain cannot impersonate the hub.

### SPIFFE ID scheme

The hub derives the OCM identity from the path of the client's SPIFFE ID, so no per-cluster
configuration is needed on the hub. The user and group names match what the `csr` and `grpc`
drivers and the add-on framework use today, so the existing RBAC created by the registration
controller and the add-on manager, and the existing identity checks, apply without change.

| Workload | SPIFFE ID | OCM user | OCM groups |
| --- | --- | --- | --- |
| Hub gRPC server | `spiffe://<td>/ocm/hub/grpc-server` | n/a (server) | n/a |
| Registration agent (`Default`, `Hosted`) | `spiffe://<td>/ocm/cluster/<cluster>/agent/registration` | `system:open-cluster-management:<cluster>:registration` | `system:open-cluster-management:<cluster>`, `system:open-cluster-management:managed-clusters` |
| Work agent (`Default`, `Hosted`) | `spiffe://<td>/ocm/cluster/<cluster>/agent/work` | `system:open-cluster-management:<cluster>:work` | same as above |
| Klusterlet agent (`Singleton`, `SingletonHosted`) | `spiffe://<td>/ocm/cluster/<cluster>/agent/klusterlet` | `system:open-cluster-management:<cluster>:klusterlet` | same as above |
| Add-on agent (all modes) | `spiffe://<td>/ocm/cluster/<cluster>/addon/<addon>/agent/<agent>` | `system:open-cluster-management:cluster:<cluster>:addon:<addon>:agent:<agent>` | `system:open-cluster-management:cluster:<cluster>:addon:<addon>`, `system:open-cluster-management:addon:<addon>` |

`<td>` is the trust domain configured on the hub. The hub rejects any SVID whose trust domain or
path does not match this scheme, any request where the cluster in the event differs from the
cluster in the SVID, and any add-on request where the add-on in the event differs from the add-on
in the SVID.

The platform must make sure SPIRE only issues `.../cluster/<cluster>/...` IDs to workloads that
really belong to cluster `<cluster>`, for example by binding registration entries to the node
attestation of the SPIRE agents that run that cluster's agents. See
[Risks and Mitigation](#risks-and-mitigation).

### Install modes

![Agent SVIDs in the four Klusterlet install modes](spiffe-grpc-registration-modes.svg)

The rule is the same in every mode: each agent gets its SVID from the local SPIRE agent, on the
node where the agent's pod runs. The hub also treats every mode the same way: it sees an SVID,
maps it, and authorizes the request. The modes differ only in where the agent pods run.

| Mode | Klusterlet agent pods run on | Klusterlet SVIDs |
| --- | --- | --- |
| `Default` | Managed cluster, `open-cluster-management-agent` | `agent/registration`, `agent/work` |
| `Singleton` | Managed cluster, `open-cluster-management-agent` | `agent/klusterlet` |
| `Hosted` | Management cluster, namespace named after the Klusterlet | `agent/registration`, `agent/work` |
| `SingletonHosted` | Management cluster, namespace named after the Klusterlet | `agent/klusterlet` |

- In every mode, each Deployment mounts the local Workload API with a `csi.spiffe.io` volume and
  has no bootstrap or hub-kubeconfig Secret volume.
- In `Singleton` and `SingletonHosted` modes the one agent pod runs registration and work and has
  a single SVID. Hub RBAC is granted to the cluster's groups, so one identity covers both roles.
- In `Hosted` and `SingletonHosted` modes one management cluster runs the agents of every cluster
  it hosts, so the SPIRE registration entries there must select on the Klusterlet's namespace:
  only the agents in namespace `c1` can get `.../cluster/c1/...` IDs.
- In `Hosted` and `SingletonHosted` modes the agents still reach the managed cluster's
  kube-apiserver with the managed-cluster kubeconfig the user provides today
  (`external-managed-kubeconfig`). That credential is for the managed cluster, not the hub, and is
  not changed by this proposal; see [Open Questions](#open-questions).
- Add-on agents run where the add-on is installed: on the managed cluster in all modes, or on the
  management cluster for hosted add-ons. Like the Klusterlet agents, they get their SVIDs from
  the local SPIRE agent.

### Lifecycle

![Hub setup, registration, renewal and revocation with SPIFFE SVIDs only](spiffe-grpc-registration-lifecycle.svg)

The steps are the same in every install mode. "Registration agent" below means the registration
agent in `Default` and `Hosted` modes, and the Klusterlet agent in the singleton modes.

**Set up the hub.**

1. The platform gives the hub gRPC server pod its identity: SPIRE agents run on the hub nodes, and a
   SPIRE registration entry maps the gRPC server pod (by namespace, ServiceAccount and label) to
   `spiffe://<td>/ocm/hub/grpc-server`. OCM does not create this entry.
2. The admin configures the `ClusterManager` with the `spiffe` driver, for example with
   `clusteradm init`.
3. The cluster-manager operator deploys the gRPC server with a `csi.spiffe.io` volume for the
   local Workload API and no Secret volumes. It creates no signing key, serving certificate,
   bootstrap ServiceAccount or token.
4. The gRPC server opens a Workload API `X509Source` and receives, in memory, its own X.509-SVID and
   the trust bundle for the trust domain, plus any federated trust domains. It reports not ready
   until the first SVID arrives.
5. It serves mTLS: it presents its SVID and verifies client SVIDs against the trust bundle. SPIRE
   pushes rotated SVIDs and bundle updates over the same stream, and new handshakes use them.
   The server also starts watching `ManagedCluster` and `ManagedClusterAddOn` for revocation.

Reading its own SVID and the trust bundle from the local Workload API is all the hub does with
SPIRE. It does not call the SPIRE server, create registration entries, mint SVIDs for agents or
publish a CA bundle for agents; agents get the trust bundle from their own local Workload API.

**Register.**

1. The registration agent fetches its X.509-SVID and trust bundle from the local Workload API and
   keeps them in memory.
2. It dials the hub gRPC endpoint from the `Klusterlet` spec over mTLS, accepting only the
   configured hub SPIFFE ID.
3. The hub verifies the client SVID against the trust bundle from its own Workload API and maps
   the SPIFFE ID to the OCM user and groups above.
4. The agent publishes a create for `ManagedCluster/<cluster>`. The hub allows it only when the
   name equals the cluster in the SVID. The new cluster has `hubAcceptsClient: false`.
5. A cluster admin or automation accepts the cluster, exactly as today. The registration
   controller creates the cluster namespace and RBAC for the cluster's groups.
6. The agent sees the accepted status and starts lease, status, ManifestWork and add-on sync.

There is no bootstrap stage. Before acceptance, the agent's identity is authorized only to create,
get and watch its own `ManagedCluster`, through a ClusterRole bound to
`system:open-cluster-management:managed-clusters`.

**Renew.**

- SPIRE rotates each SVID, by default at half of its TTL, and pushes it over the Workload API. The
  agent's TLS configuration reads the current SVID for each new handshake.
- A gRPC stream authenticates once, when the connection is established. The hub therefore records
  the client SVID's expiry for each connection and closes the connection when that SVID expires.
  The client reconnects with its current SVID and resyncs, as it already does after a reconnect.
- The hub gRPC server renews its own serving SVID the same way.
- OCM never signs, stores or rotates a certificate.

**Revoke.**

- **In OCM, immediate.** Setting `hubAcceptsClient: false` on, or deleting, the `ManagedCluster`
  makes the authorizer deny the cluster's requests. The hub gRPC server also watches
  `ManagedCluster` and closes every open connection whose SVID belongs to that cluster, including
  its add-on agents.
- **In SPIRE, within one TTL.** Deleting the cluster's registration entries, or evicting its SPIRE
  agents, stops new SVIDs. SVIDs already issued stop working when they expire. Short TTLs are the
  revocation mechanism; there is no CRL in this design.
- **Trust domain compromise.** SPIRE CA rotation and bundle updates reach OCM through the
  Workload API, with no OCM change.

**Unjoin.** Deleting the `ManagedCluster` on the hub and the `Klusterlet` on the managed or
management cluster removes everything. There are no OCM Secrets to clean up.

### Add-ons

Add-ons follow the same model. Each add-on agent has its own SVID; the registration agent no
longer obtains credentials on the add-on's behalf.

**Register.**

1. The add-on is enabled on the hub as today, by creating the `ManagedClusterAddOn`. The add-on
   manager sets the add-on's `kubeClient` registration and creates its hub RBAC from the add-on's
   permission configuration, bound to the add-on's user and groups.
2. On the managed cluster, the registration agent sees the registration and, with the `spiffe`
   add-on driver, creates no CSR, no token request and no Secret. It sets
   `status.registrations[].kubeClient.driver: spiffe` and the `RegistrationApplied` condition.
3. The add-on agent fetches its own SVID (`.../cluster/<cluster>/addon/<addon>/agent/<agent>`)
   from the local Workload API and connects to the hub gRPC server over mTLS, as the Klusterlet
   agents do.
4. The hub maps the SVID to the add-on user and groups, which are the default `kubeClient`
   subject, so the add-on's existing hub RBAC applies.

If an add-on sets a custom `kubeClient` subject that does not match the mapping for its SPIFFE ID,
the add-on manager sets `RegistrationApplied` to false with a clear reason, because no SVID can
carry that subject.

**Renew.** Exactly as for the Klusterlet agents: SPIRE rotates the add-on agent's SVID, and the
hub closes connections whose SVID has expired.

**Revoke.**

- Disabling the add-on (deleting the `ManagedClusterAddOn`) makes the add-on manager remove its
  RBAC, and the hub gRPC server closes the add-on agent's open connections for that cluster.
- Denying or deleting the cluster closes the connections of all its add-on agents.
- Deleting the add-on's SPIRE registration entries stops new SVIDs; existing ones lapse within
  their TTL.

**Add-on framework.** The add-on framework gets a helper that builds the add-on agent's
CloudEvents clients from a Workload API `X509Source`, so add-on authors do not handle SVIDs
directly.

### Implementation Details/Notes/Constraints

This section goes through each install mode in the same order: hub setup, managed cluster setup,
join, renew and revoke. `Default` is described in full. The other modes repeat the same steps
and show what changes; where a mechanism is identical, they point back to `Default`. Add-on agents
follow, in [Add-on agents](#add-on-agents), with the same steps.

**The modes at a glance**, for a Klusterlet named `<klusterlet>`:

| Mode | Agent Deployments | Where they run | SVIDs | Trust domain the cluster is pinned to |
| --- | --- | --- | --- | --- |
| `Default` | `<klusterlet>-registration-agent`, `<klusterlet>-work-agent` | Managed cluster, `open-cluster-management-agent` | `agent/registration`, `agent/work` | The managed cluster's |
| `Singleton` | `<klusterlet>-agent` | Managed cluster, `open-cluster-management-agent` | `agent/klusterlet` | The managed cluster's |
| `Hosted` | `<klusterlet>-registration-agent`, `<klusterlet>-work-agent` | Management cluster, namespace named after the Klusterlet | `agent/registration`, `agent/work` | The management cluster's |
| `SingletonHosted` | `<klusterlet>-agent` | Management cluster, namespace named after the Klusterlet | `agent/klusterlet` | The management cluster's |

The hub is set up the same way for every mode, and treats every mode the same way: it sees an
SVID, maps it and authorizes the request.

All new API fields sit behind a new `SPIFFERegistration` feature gate. Where each change lives in
the code is listed at the end, in [Changes by repository](#changes-by-repository).

#### Mode: `Default`

In `Default` mode the registration agent and the work agent run as two Deployments in
`open-cluster-management-agent` on the managed cluster. Each gets its own SVID from the SPIRE
agent on its node.

The running example has one hub and three managed clusters, each run by a team with its own
SPIRE:

| Cluster | What it is | SPIRE trust domain |
| --- | --- | --- |
| hub | The OCM hub | `ocm-hub.example.org` |
| `cluster1` | Managed cluster | `ocm-hub.example.org`, the same as the hub |
| `cluster2` | Managed cluster | `east.example.org` |
| `cluster3` | Managed cluster | `west.example.org` |

The add-on is `my-addon`, and the Klusterlets are named `klusterlet`. A trust domain is simply
"which SPIRE issued this SVID": cluster2's agents get SVIDs like
`spiffe://east.example.org/...`. A platform with a single trust domain is the same example without
cluster2 and cluster3.

##### Hub setup

**1. The platform gives the hub gRPC server its identity.** SPIRE agents run on the hub's nodes,
and a registration entry maps the gRPC server pod to
`spiffe://ocm-hub.example.org/ocm/hub/grpc-server`. OCM does not create this entry. With the SPIRE
controller manager:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-hub-grpc-server
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/hub/grpc-server"
  federatesWith:          # only when agents are in other trust domains
  - east.example.org
  - west.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-hub
  podSelector:
    matchLabels:
      app: cluster-manager-grpc-server
  workloadSelectorTemplates:
  - "k8s:sa:grpc-server-sa"
```

###### Federation

When managed clusters are in other trust domains, the SPIRE deployments must trust each other, or
the TLS handshake fails. The platform sets this up; OCM does not configure it and still talks only
to its local Workload API.

- **The hub's SPIRE trusts east and west** (`federatesWith` above), so the hub gRPC server's
  Workload API returns their trust bundles and the hub can check cluster2's and cluster3's SVIDs.
- **East's and west's SPIRE trust `ocm-hub.example.org`**, so the agents' Workload API returns
  the hub's trust bundle and cluster2 and cluster3 can check the hub's SVID (see
  [Managed cluster setup](#managed-cluster-setup)).
- With a single trust domain, nothing needs to be federated.

**2. The admin configures the `ClusterManager`.**

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
  registrationConfiguration:
    registrationDrivers:
    - authType: spiffe
      spiffe:
        # The hub's own trust domain. The hub's SVID is
        # spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        trustDomain: ocm-hub.example.org

        # Trust domains the hub accepts agents from. An agent whose SVID
        # comes from any other trust domain is refused.
        agentTrustDomains:
        - ocm-hub.example.org   # cluster1
        - east.example.org      # cluster2
        - west.example.org      # cluster3
  serverConfiguration:
    endpointsExposure:
    - protocol: grpc-spiffe   # the SPIFFE listener; see "gRPC server listeners"
      grpc:
        type: loadBalancer    # must pass TLS through; see Risks
```

```shell
clusteradm init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org \
  --spiffe-agent-trust-domains ocm-hub.example.org,east.example.org,west.example.org
```

Only `trustDomain` is required. If every cluster is in the hub's trust domain, leave out
`agentTrustDomains` (and `--spiffe-agent-trust-domains`): it defaults to just the hub's own trust
domain.

| Field | Default | Purpose |
| --- | --- | --- |
| `trustDomain` | none (required) | The hub's trust domain. The hub gRPC server's SVID is in it. |
| `agentTrustDomains` | `[<trustDomain>]` | Trust domains whose Klusterlet and add-on agents the hub accepts. An SVID from any other trust domain is refused. Each managed cluster is pinned to one of them; see [Trust domain pinning](#trust-domain-pinning). |
| `serverSPIFFEID` | `spiffe://<trustDomain>/ocm/hub/grpc-server` | The SVID the hub gRPC server presents. |
| `workloadAPISocket` | `unix:///spiffe-workload-api/spire-agent.sock` | The local SPIRE agent's Workload API. |
| `maxSVIDLifetime` | `24h` | Must be at least the longest SVID TTL SPIRE issues to OCM agents, in any of `agentTrustDomains`. The delete guard waits this long; see [Deleting a cluster](#deleting-a-cluster). |
| `maxClockSkew` | `5m` | Allowed clock difference between the hub and the SPIRE servers of every trust domain. Used by the SVID validity check and added to the delete guard. |

A hub with a single trust domain, with every field set to its default:

```yaml
      spiffe:
        trustDomain: ocm-hub.example.org
        agentTrustDomains:
        - ocm-hub.example.org
        serverSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        workloadAPISocket: unix:///spiffe-workload-api/spire-agent.sock
        maxSVIDLifetime: 24h
        maxClockSkew: 5m
```

**3. The cluster-manager operator deploys the gRPC server.** It renders a SPIFFE version of the
gRPC server Deployment. What the gRPC server gets:

| Today | In `spiffe` mode |
| --- | --- |
| The hub signing key (`signer-secret`) | Nothing |
| The server's TLS certificate, in a Secret | Nothing |
| A bootstrap ServiceAccount and token | Nothing |
| | A `csi.spiffe.io` volume, which puts the local SPIRE agent's Workload API socket into the pod |

For example, with the default `workloadAPISocket`:

```yaml
    spec:
      serviceAccountName: grpc-server-sa
      containers:
      - name: grpc-server
        volumeMounts:
        - name: spiffe-workload-api
          mountPath: /spiffe-workload-api
          readOnly: true
      volumes:
      - name: spiffe-workload-api
        csi:
          driver: csi.spiffe.io
          readOnly: true
```

The no-key-material rule covers every hub credential: agent and add-on identities toward the hub,
and the hub gRPC server's own identity.

**4. The gRPC server gets its SVID and starts serving.**

- **It picks its own SVID by ID.** The Workload API can return several SVIDs for one pod, for
  example when the platform also gives every pod a generic identity, and `X509Source` uses the
  first one by default. So every OCM component opens its source with
  `workloadapi.WithDefaultX509SVIDPicker` and picks the SVID whose SPIFFE ID it expects, never by
  position. The gRPC server expects the configured `serverSPIFFEID`. If no SVID with that ID is
  present, it reports not ready, with the IDs it did receive, instead of presenting another
  identity.
- **It sets up mTLS from the Workload API** (go-spiffe `workloadapi.X509Source` and
  `tlsconfig.MTLSServerConfig`) instead of certificate files. The source holds the hub's trust
  bundle and the federated ones, and the server's authorizer accepts only client SVIDs whose trust
  domain is in `agentTrustDomains`, even if the hub's SPIRE federates with more:

  ```go
  tlsconfig.MTLSServerConfig(source, source, authorizeMemberOfAny(agentTrustDomains))
  ```

- **It watches** `ManagedCluster` and `ManagedClusterAddOn`, for revocation (see
  [Revoke](#revoke)).

###### gRPC server listeners

The `spiffe` driver gets its own listener, on its own port and Service
(`cluster-manager-grpc-server-spiffe`), exposed through an `endpointsExposure` entry with
`protocol: grpc-spiffe`. That listener uses only the SPIFFE TLS credentials and the SPIFFE
authenticator (no tokens, no OCM-signed certificates) and does not offer the CSR and token-request
services.

| `registrationDrivers` | Listeners | Hub Secrets |
| --- | --- | --- |
| `spiffe` only | Only the SPIFFE listener. | None: the operator creates no signer, serving certificate or bootstrap token. |
| `spiffe` and `grpc`, for migration | The existing `grpc` listener keeps running unchanged on its current port, with its current authentication and its CSR and token-request services, next to the SPIFFE listener. Agents that have not migrated keep working. | The `grpc` driver's Secrets, as today. |

Example: a hub in the middle of moving its clusters from `grpc` to `spiffe`:

```yaml
spec:
  registrationConfiguration:
    registrationDrivers:
    - authType: grpc
    - authType: spiffe
      spiffe:
        trustDomain: ocm-hub.example.org
  serverConfiguration:
    endpointsExposure:
    - protocol: grpc          # existing listener, unchanged
      grpc:
        type: loadBalancer
    - protocol: grpc-spiffe   # new listener, Service cluster-manager-grpc-server-spiffe
      grpc:
        type: loadBalancer
```

Both listeners share the same CloudEvents services and authorizer, so a cluster sees the same
resources whichever one it uses. When `grpc` is removed from `registrationDrivers`, its listener
and its Secrets are removed.

**In short:** the hub gRPC server has an SVID from SPIRE, no Secrets, and accepts agents only from
the trust domains listed in `agentTrustDomains`.

##### Managed cluster setup

**1. The platform gives each agent its identity.** SPIRE agents run on the managed cluster's
nodes, and one registration entry per agent maps its pod to its SPIFFE ID. OCM does not create
these entries.

Entries pick pods by properties such as namespace and ServiceAccount. That is not enough here: the
singleton agent and the default-mode work agent both use the ServiceAccount `<klusterlet>-work-sa`,
so an entry could not tell them apart. So every agent pod gets a label,
`open-cluster-management.io/spiffe-role`, with the value `registration`, `work` or `klusterlet`. A
label alone is not enough either, because anyone who can create a pod in the namespace can set it.
Entries must therefore select on the namespace, the ServiceAccount and the role label together.

For the work agent of cluster1:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-work-agent
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"   # SPIRE also checks the pod's ServiceAccount
```

For cluster2, in `east.example.org`, the entry is in east's trust domain and federates with the
hub's:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-work-agent
spec:
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster2/agent/work"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

The registration agent's entry is the same with `agent/registration`, `spiffe-role: registration`
and `k8s:sa:klusterlet-registration-sa`.

**2. The admin configures the `Klusterlet`.** It tells the agents where the hub is, which SPIFFE
ID the hub must present and, if it differs from the hub's, their own trust domain. None of this is
secret, so it goes in the spec instead of a bootstrap Secret. Add-ons are switched to SPIFFE with
the existing `addOnKubeClientRegistrationDriver` field.

cluster1 is in the hub's trust domain, so it needs nothing extra:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster1
  deployOption:
    mode: Default
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        workloadAPISocket: unix:///spiffe-workload-api/spire-agent.sock
    addOnKubeClientRegistrationDriver:
      authType: spiffe       # required when registrationDriver.authType is spiffe
```

cluster2 is in `east.example.org`, so it says so with `trustDomain`:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster2
  deployOption:
    mode: Default
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        # Still the hub's ID, in the hub's trust domain
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        # The agents' own trust domain
        trustDomain: east.example.org
    addOnKubeClientRegistrationDriver:
      authType: spiffe
```

cluster3 is the same as cluster2, with `clusterName: cluster3` and `trustDomain: west.example.org`.

| Field | Default | Purpose |
| --- | --- | --- |
| `hubEndpoint` | none (required) | The hub gRPC server's SPIFFE listener, `host:port`. |
| `hubSPIFFEID` | none (required) | The only SPIFFE ID the agents accept from the hub. |
| `trustDomain` | The trust domain of `hubSPIFFEID` | The agents' own trust domain. Agents build their expected SPIFFE IDs in it. |
| `workloadAPISocket` | `unix:///spiffe-workload-api/spire-agent.sock` | The local SPIRE agent's Workload API. |

The same fields work in all four install modes; only `deployOption.mode` changes.

**3. The klusterlet operator checks the cluster and installs the admission policy** (see
[Admission policy](#admission-policy) and [Requirements](#requirements)). If either check fails, it
deploys no agent.

**4. The klusterlet operator deploys the agents.** It renders a SPIFFE version of each agent
Deployment. What the agents get:

| Today | In `spiffe` mode |
| --- | --- |
| A `bootstrap-hub-kubeconfig` Secret to start from | Nothing |
| A `hub-kubeconfig-secret` the registration agent writes its credentials to | Nothing |
| | The same `csi.spiffe.io` volume as the hub gRPC server, in each agent Deployment; each agent fetches its own SVID |

| Deployment | ServiceAccount | `spiffe-role` | SVID for cluster1 |
| --- | --- | --- | --- |
| `klusterlet-registration-agent` | `klusterlet-registration-sa` | `registration` | `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/registration` |
| `klusterlet-work-agent` | `klusterlet-work-sa` | `work` | `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work` |

###### Admission policy

**The attack.** Selecting on namespace, ServiceAccount and label only helps if nobody else can
start a pod with that combination. Restricting Pod creation alone is not enough: anyone who can
create a Deployment, ReplicaSet, StatefulSet, DaemonSet, Job or CronJob in the namespace can put
the agent's ServiceAccount and `spiffe-role` label in the pod template, and a Kubernetes
controller then creates the Pod for them. For example, a user allowed to create Jobs in
`open-cluster-management-agent` could run a Job as `klusterlet-work-sa` with `spiffe-role: work`
and receive cluster1's work-agent SVID.

**What OCM does.** In `spiffe` mode the Klusterlet operator installs a ValidatingAdmissionPolicy
and binding that cover the agent namespaces (`open-cluster-management-agent`, and the
Klusterlet-named namespaces on a management cluster):

| Object | Create | Change after creation |
| --- | --- | --- |
| Deployment | Only the Klusterlet operator's ServiceAccount. | Only the Klusterlet operator's ServiceAccount. |
| ReplicaSet | Only the Deployment controller, with a Deployment as owner. | The spec only by the Deployment controller, and the Pod template not at all. The Deployment controller only scales ReplicaSets; for a new template it creates a new ReplicaSet. Without this, someone allowed to update ReplicaSets could change the image, and the ReplicaSet controller would create Pods from it that pass every other check. |
| Pod | Only the ReplicaSet controller, with a ReplicaSet as owner. | Nobody may change its container images or add ephemeral containers (the `pods/ephemeralcontainers` subresource). Kubernetes allows image updates on a running Pod, so without this a user could swap the code inside an agent Pod that already holds an SVID. Status updates, and the label and annotation changes controllers make, are still allowed. |
| StatefulSet, DaemonSet, Job, CronJob | Refused. | Refused. |

So every Pod traces back to a Deployment, and only the operator can create Deployments.

A Pod's `spiffe-role` label must also match its ServiceAccount, and neither may change after
creation. These are the only allowed pairs, for a Klusterlet named `<klusterlet>`:

| Install mode | `spiffe-role` | ServiceAccount |
| --- | --- | --- |
| `Default`, `Hosted` | `registration` | `<klusterlet>-registration-sa` |
| `Default`, `Hosted` | `work` | `<klusterlet>-work-sa` |
| `Singleton`, `SingletonHosted` | `klusterlet` | `<klusterlet>-work-sa` |

The singleton agent reuses `<klusterlet>-work-sa`, so the label is what tells it apart from a work
agent. A pod with `<klusterlet>-work-sa` and any other role label is refused.

What the policy does with each request on cluster1:

| Request | Result |
| --- | --- |
| The operator creates Deployment `klusterlet-work-agent`; the Deployment controller creates its ReplicaSet; the ReplicaSet controller creates a Pod with `klusterlet-work-sa` and `spiffe-role: work` | Admitted |
| A user creates a Job that runs as `klusterlet-work-sa` with `spiffe-role: work` | Refused |
| A user creates a Deployment in `open-cluster-management-agent` | Refused |
| A user changes the image in a ReplicaSet's Pod template, or of a running agent Pod | Refused |
| A Pod with `klusterlet-work-sa` and `spiffe-role: klusterlet` | Refused: `klusterlet` is a singleton role |

Namespace RBAC that grants these permissions only to the operator is the first layer; the policy
holds even if RBAC is later widened by mistake.

Add-on agent namespaces get the same policy, with a few differences; see
[Add-on agents](#add-on-agents).

###### How admission checks the chain

**The problem.** A ValidatingAdmissionPolicy only sees the request in front of it: the user making
it and the object. When the ReplicaSet controller creates a Pod, the policy sees that controller,
not whoever created the Deployment, and it cannot look that up.

**What the policy does.** It checks the creator once, at the top of the chain, and every step below
by who makes the request and the owner kind. Following cluster1's work agent down the chain:

| Step | Request comes from | Policy checks |
| --- | --- | --- |
| Deployment (or, in add-on namespaces, DaemonSet or StatefulSet) created or changed | The real user | The user is the operator or the work agent. |
| ReplicaSet created | `system:serviceaccount:kube-system:deployment-controller` | The requester is the Deployment controller, and the owner is a Deployment. |
| Pod created | The matching built-in controller | The requester is that controller, and the owner is a ReplicaSet, DaemonSet or StatefulSet. |

While the policy is in place, nobody else can create the top object, so every pod traces back to
the operator or the work agent. Workloads created while the policy was missing skip that first
check; they are handled by the recovery in [If the policy is removed](#if-the-policy-is-removed).

###### Requirements

- **Separate controller identities.** The chain check needs each built-in controller to run under
  its own ServiceAccount, which is what kube-controller-manager's
  `--use-service-account-credentials=true` does. kubeadm sets it by default; on managed services
  the platform must confirm it. If the controllers share one identity instead, the policy refuses
  their ReplicaSets and Pods, so no agent starts and the Klusterlet reports `Degraded`. That is
  safe, but unsupported: `spiffe` mode requires separate controller ServiceAccounts.
- **The policy is required, not optional.** ValidatingAdmissionPolicy is GA from Kubernetes 1.30,
  so `spiffe` mode needs Kubernetes 1.30 or later on every cluster that runs OCM agents: the
  managed cluster, or the management cluster in hosted modes. Before it deploys any agent, the
  Klusterlet operator checks that the `admissionregistration.k8s.io/v1` ValidatingAdmissionPolicy
  API is served and that its policy and binding exist. If either check fails, it deploys no agent
  and sets a `Degraded` condition on the Klusterlet that says why. The operator installs the policy
  and binding itself, so on every reconcile it also recreates them if they are missing. If they
  are removed, it runs the recovery steps below before any agent comes back.

###### If the policy is removed

**The accepted gap.** Scaling to zero stops the real agents, but it does not close every hole. The
policy and its binding are cluster-scoped, so only cluster-admin (and the operator) can delete
them. If a cluster-admin deletes them *and* namespace RBAC also lets someone else create workloads
in an agent namespace, that user can start a pod with the agent's ServiceAccount and role label
before the operator reacts, and SPIRE gives it an SVID. This needs two mistakes at once.

**Recovery** runs in the operator, in this order:

1. Scale all agents to zero, including the work agent.
2. Recreate the policy and binding.
3. Delete every Deployment, ReplicaSet, DaemonSet, StatefulSet and Pod in the Klusterlet agent
   namespaces and the add-on namespaces, trusted or not. Someone with widened RBAC could have
   created new workloads or changed existing ones, and the operator cannot tell a changed workload
   from a clean one, so it removes them all.
4. Recreate its own agent Deployments and scale them up. The work agent then re-applies the add-on
   workloads from their `ManifestWork`s, so every add-on workload comes back as a clean copy of
   what the hub defined, and passes the restored policy.

The restored policy alone cannot refuse new pods from a workload created during the gap, because
those pods come from the normal built-in controller (see
[How admission checks the chain](#how-admission-checks-the-chain)). So the gap lasts until step 3
finishes, not just until the policy is back.

- The operator watches the policy and binding and runs these steps in one reconcile, so the gap is
  normally short.
- If step 2 fails, for example because something keeps deleting the policy, the agents stay at
  zero, the gap stays open and the Klusterlet stays `Degraded`.
- Add-ons are briefly down during recovery.
- A pod that got an SVID during the gap can still use it until it expires, and the hub can stop it
  sooner only by denying or deleting the cluster.

We accept this rather than tying SPIRE entries to the policy, because OCM does not manage SPIRE
entries.

**In short:** each agent Deployment mounts the Workload API and has no Secrets, and only pods that
trace back to the operator, or to a manifest the hub defined, can get an agent's SVID.

##### Join

There is no bootstrap stage. Before acceptance, the agent's identity is authorized only to create,
get and watch its own `ManagedCluster`, through a ClusterRole bound to
`system:open-cluster-management:managed-clusters`.

```shell
# cluster1, in the hub's trust domain: no token
clusteradm join --cluster-name cluster1 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server

# cluster2: add the agents' own trust domain (cluster3 is the same with west.example.org)
clusteradm join --cluster-name cluster2 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server \
  --spiffe-trust-domain east.example.org
```

What happens when cluster2 joins:

1. **The registration agent picks its SVID.** It gets
   `spiffe://east.example.org/ocm/cluster/cluster2/agent/registration` from its local SPIRE, and
   keeps it in memory. It picks it by ID, built from the Klusterlet's trust domain, cluster name
   and role, as the hub does (see [Hub setup](#hub-setup)). If the platform also gives the pod a
   generic SVID such as
   `spiffe://east.example.org/ns/open-cluster-management-agent/sa/klusterlet-registration-sa`, the
   agent ignores it; if the OCM ID is missing, it reports not ready, with the IDs it did
   receive.
2. **It connects to the hub and checks the hub's ID.** It dials the `hubEndpoint` over mTLS and
   accepts only the configured hub ID:

   ```go
   tlsconfig.MTLSClientConfig(source, source, tlsconfig.AuthorizeID(hubSPIFFEID))
   ```

   A server that presents any other ID, even one from the same trust domain, is refused. The
   source verifies the hub's SVID with the hub's trust bundle, which cluster2's Workload API
   returns through federation.
3. **The hub checks the agent's SVID and maps it** (see
   [Mapping SPIFFE IDs to OCM identities](#mapping-spiffe-ids-to-ocm-identities)).
   `east.example.org` is in `agentTrustDomains`, so the agent is let in, as the user
   `system:open-cluster-management:cluster2:registration`.
4. **The agent asks to create `ManagedCluster/cluster2`.** The hub allows it only when the name
   equals the cluster in the SVID. The gRPC server's cluster service creates it, with
   `hubAcceptsClient: false`, and records which trust domain it came from, replacing any value the
   agent put in the request:

   ```yaml
   apiVersion: cluster.open-cluster-management.io/v1
   kind: ManagedCluster
   metadata:
     name: cluster2
     annotations:
       open-cluster-management.io/spiffe-trust-domain: east.example.org   # set by the hub
   spec:
     hubAcceptsClient: false
   ```

5. **The hub admin accepts it.** The admin sees `east.example.org`; accepting the cluster means
   trusting that trust domain for that cluster. `--spiffe-trust-domain` refuses to accept a cluster
   pinned to any other trust domain:

   ```shell
   clusteradm accept --clusters cluster1
   clusteradm accept --clusters cluster2 --spiffe-trust-domain east.example.org
   ```

   The registration controller creates the cluster namespace and RBAC for the cluster's groups,
   exactly as today.
6. **The agents start working.** The registration agent sees the accepted status and starts lease,
   status, ManifestWork and add-on sync. The work agent connects the same way with its own SVID,
   `spiffe://east.example.org/ocm/cluster/cluster2/agent/work`, and syncs ManifestWorks.

On the agent side this is the new registration driver: it builds the agent's gRPC clients from its
SVID. It has no CSR step, no Secret handling and no bootstrap phase: the agent can talk to the hub
as long as it has a current SVID and the cluster is accepted. The registration agent does not
start its Secret-based hub-kubeconfig controller, and the work agent fetches its own SVID instead
of reading a Secret written by the registration agent.

###### Mapping SPIFFE IDs to OCM identities

A new `SPIFFEAuthenticator` reads the client's verified SVID and turns it into an OCM identity:

1. It rejects IDs from a trust domain that is not in `agentTrustDomains`, and IDs that do not
   follow the scheme.
2. It turns the SPIFFE ID's path into an OCM user and groups.
3. It adds `system:authenticated` (see below).
4. It also returns the SVID's trust domain, because the trust domain is not part of the user name,
   and the hub checks it against the cluster's pinned trust domain (see
   [Trust domain pinning](#trust-domain-pinning)).
5. It notes when the SVID expires.

| SVID | User | Groups |
| --- | --- | --- |
| `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work` | `system:open-cluster-management:cluster1:work` | `system:open-cluster-management:cluster1`, `system:open-cluster-management:managed-clusters`, `system:authenticated` |
| `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet` | `system:open-cluster-management:cluster1:klusterlet` | the same three groups |
| `spiffe://east.example.org/ocm/cluster/cluster2/agent/work` | `system:open-cluster-management:cluster2:work` | `system:open-cluster-management:cluster2`, `system:open-cluster-management:managed-clusters`, `system:authenticated` |
| `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/addon/my-addon/agent/my-addon-agent` | `system:open-cluster-management:cluster:cluster1:addon:my-addon:agent:my-addon-agent` | `system:open-cluster-management:cluster:cluster1:addon:my-addon`, `system:open-cluster-management:addon:my-addon`, `system:authenticated` |

| Rejected SVID | Why |
| --- | --- |
| `spiffe://other.example.org/ocm/cluster/cluster1/agent/work` | `other.example.org` is not in `agentTrustDomains` |
| `spiffe://ocm-hub.example.org/ocm/something-else` | Does not follow the scheme |

**Why `system:authenticated`.** The kube-apiserver gives every authenticated user that group
today. The hub sends exactly these groups in its SubjectAccessReviews, so without it a permission
bound to `system:authenticated`, which works for an agent today, would be denied after the switch
to `spiffe`.

###### Trust domain pinning

The hub has one trust domain, for its own SVID. Managed clusters can be in that trust domain or in
others, for example one per region or per platform team. In the
[SPIFFE ID scheme](#spiffe-id-scheme), `<td>` is then the hub's trust domain for the hub gRPC
server, and the cluster's own trust domain for its Klusterlet and add-on agents.

**Why pinning is needed.** Look at the user name in step 3 of the join:
`system:open-cluster-management:cluster2:registration`. The OCM user and group names come from the
path of the SPIFFE ID, not from its trust domain, so the existing RBAC applies whatever the trust
domain. But it also means that, without a further check, both of these SVIDs would become the same
user:

- `spiffe://east.example.org/ocm/cluster/cluster2/agent/work`: the real cluster2.
- `spiffe://west.example.org/ocm/cluster/cluster2/agent/work`: issued by west's SPIRE, by mistake
  or by an attacker.

West could then take over cluster2. The annotation `open-cluster-management.io/spiffe-trust-domain`
stops that: the hub refuses any request about a cluster from an SVID in a different trust domain
than the cluster's annotation, for the Klusterlet agents and the cluster's add-on agents alike.

| Who connects | Result |
| --- | --- |
| `spiffe://east.example.org/ocm/cluster/cluster2/agent/work` | Allowed: cluster2 belongs to east |
| `spiffe://east.example.org/ocm/cluster/cluster2/addon/my-addon/agent/my-addon-agent` | Allowed: cluster2's add-ons are also from east |
| `spiffe://west.example.org/ocm/cluster/cluster2/agent/work` | Refused: cluster2 belongs to east, not west |
| `spiffe://west.example.org/ocm/cluster/cluster3/agent/work` | Allowed: cluster3 belongs to west |
| `spiffe://west.example.org/ocm/cluster/cluster4/agent/work`, no `cluster4` yet | May create `cluster4`, pinned to west and not accepted |
| `spiffe://south.example.org/ocm/cluster/cluster4/agent/work` | Refused: `south.example.org` is not in `agentTrustDomains` |

**Who can change the annotation.** On agent updates, the cluster service keeps the annotation's
current value, as it already does for the cluster-set label. Agents have no access to the hub
kube-apiserver in this mode, so only hub users allowed to update `ManagedCluster`s can change it,
for example to move a cluster to another trust domain.

**A cluster without the annotation**, for example one created before the switch from the `grpc`
driver, or pre-created by an admin:

- If `agentTrustDomains` has one entry, the cluster is treated as pinned to it.
- If it has several, the hub refuses the cluster's SPIFFE requests until a hub admin sets the
  annotation.

**In short:** `agentTrustDomains` decides which SPIREs the hub trusts at all, and the annotation
decides which one SPIRE may speak for each cluster.

Once the cluster is accepted, add-ons can be enabled on it; see [Add-on agents](#add-on-agents).

**In short:** each agent joins with its own SVID; the hub creates the `ManagedCluster` pinned to
the SVID's trust domain, and the admin accepts it as today.

##### Renew

SPIRE renews every SVID; OCM never signs, stores or rotates a certificate.

1. **SPIRE rotates the SVIDs.** By default at half of its TTL, SPIRE pushes a new SVID for each
   agent over the Workload API. The agent's TLS configuration reads the current SVID for each new
   handshake.
2. **The hub closes connections whose SVID has expired** (see below), and the agent reconnects
   with its current SVID and resyncs, as it already does after a reconnect.
3. **The hub gRPC server renews its own serving SVID the same way**, and SPIRE pushes trust bundle
   updates, including federated ones and SPIRE CA rotations, over the same stream. New handshakes
   use them, with no OCM change.
4. **Add-on agents renew the same way**; see [Add-on agents](#add-on-agents).

###### Server TLS and SVID expiry

**The problem.** A gRPC connection checks the client's identity only once, when it opens, and
then stays open. Without more, an agent could keep using a connection long after the SVID it
opened it with has expired.

**What the hub does.** It enforces SVID expiry on the connection, not the stream:

1. A gRPC `TransportCredentials` wrapper around the SPIFFE TLS credentials records the peer's
   SPIFFE ID and SVID expiry after each handshake.
2. It registers the underlying `net.Conn` in a connection registry and arms a timer that closes the
   `net.Conn` when the SVID expires. That ends the HTTP/2 connection and every stream and unary
   call on it.
3. As a second check, unary and stream interceptors reject any RPC whose peer SVID has already
   expired.

Example:

| Time | What happens |
| --- | --- |
| 10:40 | cluster1's work agent connects with an SVID valid until 11:00. The hub arms a timer for 11:00. |
| 11:00 | The hub closes that connection. |
| Right after | The agent opens a new connection, with a new handshake, using its newer SVID, and resyncs. |

The connection registry is also how the hub closes connections when a cluster or add-on is
revoked; see [Revoke](#revoke).

**In short:** SPIRE rotates, the agents pick up the new SVID on the next handshake, and the hub
makes sure no connection outlives the SVID it was opened with.

##### Revoke

| The admin does this on the hub | What happens |
| --- | --- |
| Sets `hubAcceptsClient: false` on `cluster2`, or deletes `cluster2` | The authorizer denies the cluster's requests, and the hub closes every connection whose SVID belongs to `cluster2`, including its add-on agents. |
| Removes `east.example.org` from `agentTrustDomains` | The hub refuses its SVIDs and closes the open connections of every cluster pinned to it. The clusters become unavailable; their `ManagedCluster`s are not deleted. |

The gRPC server watches `ManagedCluster` and `ManagedClusterAddOn` and closes the matching
connections right away through the connection registry described under
[Server TLS and SVID expiry](#server-tls-and-svid-expiry). The agents reconnect, and the hub refuses
them. Disabling a single add-on is covered under [Add-on agents](#add-on-agents).

In SPIRE, deleting a cluster's registration entries, or evicting its SPIRE agents,
stops new SVIDs, and the ones already issued stop working within their TTL.

###### Deleting a cluster

**The problem.** Closing connections does not cancel an SVID that is still valid. Say an admin
deletes `c1` at 10:40, and its agent still holds an SVID valid until 11:20. Without a guard, the
agent reconnects and uses its pre-acceptance permission to create the `ManagedCluster` again.

**What the hub does.**

1. It uses one upper bound for every deletion: `spiffe.maxSVIDLifetime` from the `ClusterManager`
   (default `24h`). It must be at least as long as the longest SVID TTL SPIRE issues to OCM
   agents, in any of the agent trust domains. The hub does not try to learn the lifetime from the
   SVIDs it sees, because an agent can hold a newer, longer-lived SVID that it has not shown to the
   hub yet, for example after the platform raises the TTL.
2. In `spiffe` mode each `ManagedCluster` gets the finalizer
   `open-cluster-management.io/spiffe-revocation`. When the cluster is deleted, the hub keeps the
   finalizer until the deletion time plus `maxSVIDLifetime` plus `maxClockSkew` (default `5m`) has
   passed.
   - The skew term covers a SPIRE server whose clock is ahead of the hub's: an SVID it issued just
     before the deletion can expire up to that much later by the hub's clock.
   - By then every SVID issued before the deletion has expired, including newer ones the agent
     received but has not yet shown to the hub.
   - While the object is terminating, a create fails because the name still exists, and the
     authorizer denies every request from the cluster's identities.
3. Then the finalizer is removed and the object goes away, with its trust domain annotation. If
   the cluster registers again, it is pinned afresh and must be accepted again.

Example: SVIDs live 1 hour, `maxSVIDLifetime` is set to `1h`, `maxClockSkew` is `5m`, and `c1` is
deleted at 10:40.

| Time | State of `c1` |
| --- | --- |
| 10:40 | Deleted. The finalizer keeps it terminating; every request from its identities is denied, and it cannot be recreated. |
| 11:45 | 10:40 + `1h` + `5m`. The finalizer is removed and the object goes away. |
| After 11:45 | The agent can register again only with an SVID issued after the deletion, which means SPIRE is still issuing it identities. |

With the default `24h`, the wait is longer but just as safe.

**Keeping a cluster out for good.** The platform must remove the cluster's SPIRE registration
entries *before* it deletes the `ManagedCluster`, and also remove what creates them:

1. Delete or change the cluster's `ClusterSPIFFEID` objects (or whatever else creates its
   entries), and its registration entries.
2. Then delete the `ManagedCluster`.

With the SPIRE controller manager, deleting only the generated entries is not enough, because the
controller manager recreates them on its next reconcile. Done in this order, the last SVID SPIRE
issued for the cluster is older than the deletion time, and the wait covers it. If the entries
were removed later, for example near the end of the wait, an SVID issued just before removal could
still be valid after the finalizer goes and could recreate the cluster. OCM cannot check the
order, because it does not talk to the SPIRE server, so this is a documented step in the cluster
removal procedure.

**Unjoin.** Deleting the `ManagedCluster` on the hub and the `Klusterlet` on the managed cluster
removes everything. There are no OCM Secrets to clean up.

**In short:** denying or deleting a cluster cuts it off at once, and the hub never lets a deleted
cluster back until every SVID it could hold has expired.

#### Mode: `Singleton`

In `Singleton` mode one Deployment, `<klusterlet>-agent`, runs registration and work in a single
pod, with a single SVID. The example is cluster1, with a Klusterlet named `klusterlet`.

##### Hub setup

The same as [Default](#hub-setup). Nothing on the hub depends on the install mode.

##### Managed cluster setup

**1. The platform's SPIRE entry** is for the one agent. It uses the work agent's ServiceAccount
with the role `klusterlet`:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-klusterlet-agent
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: klusterlet
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

There are no `agent/registration` or `agent/work` entries in this mode.

**2. The `Klusterlet`** is the same as in `Default`, with `deployOption.mode: Singleton`.

**3. The klusterlet operator** checks the cluster and installs the same admission policy as in
`Default` (see [Admission policy](#admission-policy)). Its only allowed pair is `klusterlet` with
`klusterlet-work-sa`, so a pod with `spiffe-role: work` is refused in this mode.

**4. It deploys one agent**, with the same CSI volume and no Secrets:

| Deployment | ServiceAccount | `spiffe-role` | SVID for cluster1 |
| --- | --- | --- | --- |
| `klusterlet-agent` | `klusterlet-work-sa` | `klusterlet` | `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet` |

The singleton agent reuses `klusterlet-work-sa`, so the label is what tells it apart from a work
agent.

##### Join

```shell
clusteradm join --cluster-name cluster1 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server \
  --singleton
clusteradm accept --clusters cluster1
```

The steps are the same as in [Default](#join), with one agent instead of two:

1. The agent picks `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet`.
2. It connects and checks the hub's ID.
3. The hub maps it to `system:open-cluster-management:cluster1:klusterlet`, with the groups
   `system:open-cluster-management:cluster1`, `system:open-cluster-management:managed-clusters` and
   `system:authenticated`.
4. It creates `ManagedCluster/cluster1`, pinned to the SVID's trust domain.
5. The admin accepts it.
6. The same agent then does both registration and work. Hub RBAC is granted to the cluster's
   groups, so one identity covers both roles.

Add-ons: see [Add-on agents](#add-on-agents).

##### Renew

The same as in [Default](#renew), with one SVID: SPIRE rotates `agent/klusterlet`, and the hub
closes the agent's connections when that SVID expires. Registration and work traffic both
reconnect with the new SVID.

##### Revoke

The same as in [Default](#revoke). Denying or deleting cluster1 closes the klusterlet agent's
connections and its add-on agents'. To keep the cluster out for good, the platform removes the
`ocm-klusterlet-agent` entry before deleting the `ManagedCluster`.

#### Mode: `Hosted`

In `Hosted` mode the registration agent and the work agent run on a management cluster, in a
namespace named after the Klusterlet. One management cluster can host the agents of many managed
clusters.

The example is a management cluster, `mgmt1`, whose SPIRE is in `east.example.org`, hosting two
managed clusters, `cluster4` and `cluster5`. Each Klusterlet is named after its cluster, so
cluster4's agents run in namespace `cluster4` on mgmt1 and use the ServiceAccounts
`cluster4-registration-sa` and `cluster4-work-sa`.

##### Hub setup

The same as in [Default](#hub-setup). The management cluster's trust domain must be in
`agentTrustDomains`, here `east.example.org`.

##### Managed cluster setup

The agents run on the management cluster, so everything identity-related happens there.

**1. The platform's SPIRE entries are on the management cluster.** Because one management cluster
runs the agents of every cluster it hosts, the entries must select on the Klusterlet's namespace:
only the agents in namespace `cluster4` can get `.../cluster/cluster4/...` IDs. For cluster4's work
agent, on mgmt1:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster4-work-agent
spec:
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster4/agent/work"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: cluster4
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:cluster4-work-sa"
```

cluster5 has the same entries for namespace `cluster5`, so a pod in `cluster5` can never get an ID
for cluster4.

**2. The `Klusterlet`** goes on the management cluster, with `deployOption.mode: Hosted` and the
management cluster's trust domain:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: cluster4
spec:
  clusterName: cluster4
  deployOption:
    mode: Hosted
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        trustDomain: east.example.org   # the management cluster's
    addOnKubeClientRegistrationDriver:
      authType: spiffe
```

The agents still reach the managed cluster's kube-apiserver with the managed-cluster kubeconfig the
user provides today (`external-managed-kubeconfig`), and the operator still derives managed-cluster
kubeconfig Secrets from it, exactly as today. Those Secrets authenticate to the managed cluster,
not the hub, and stay out of scope until Open Question 5 is resolved.

**3. The klusterlet operator** installs the same admission policy on the management cluster,
covering the Klusterlet-named namespaces (see [Admission policy](#admission-policy)). The allowed
pairs are the same as in `Default`. The management cluster needs Kubernetes 1.30 or later and
separate controller ServiceAccounts (see [Requirements](#requirements)).

**4. It deploys the agents** in namespace `cluster4` on mgmt1, with the CSI volume and no hub
Secrets:

| Deployment | ServiceAccount | `spiffe-role` | SVID for cluster4 |
| --- | --- | --- | --- |
| `cluster4-registration-agent` | `cluster4-registration-sa` | `registration` | `spiffe://east.example.org/ocm/cluster/cluster4/agent/registration` |
| `cluster4-work-agent` | `cluster4-work-sa` | `work` | `spiffe://east.example.org/ocm/cluster/cluster4/agent/work` |

##### Join

```shell
clusteradm join --cluster-name cluster4 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server \
  --spiffe-trust-domain east.example.org \
  --mode hosted --managed-cluster-kubeconfig cluster4.kubeconfig
clusteradm accept --clusters cluster4 --spiffe-trust-domain east.example.org
```

The steps are the same as in [Default](#join). The SVIDs come from the management cluster's SPIRE,
so cluster4 is pinned to the management cluster's trust domain, `east.example.org`, not to whatever
trust domain the managed cluster itself may use.

Add-ons: see [Add-on agents](#add-on-agents), which also covers where add-on agents run in the
hosted modes and which trust domain they must be in.

##### Renew

The same as in [Default](#renew). The management cluster's SPIRE rotates the agents' SVIDs.

##### Revoke

The same as in [Default](#revoke). To keep cluster4 out for good, the platform removes its entries
on the management cluster, and the `ClusterSPIFFEID` objects that create them, before deleting the
`ManagedCluster`. Unjoin deletes the `ManagedCluster` on the hub and the `Klusterlet` on the
management cluster.

A fully compromised management cluster can act for the clusters it hosts, but that is already true
today, because it holds their hub credentials.

#### Mode: `SingletonHosted`

`SingletonHosted` combines the two: one agent, `<klusterlet>-agent`, with one SVID, on the
management cluster, in a namespace named after the Klusterlet. The example is cluster5, hosted on
mgmt1 in `east.example.org`.

##### Hub setup

The same as in [Default](#hub-setup). The management cluster's trust domain must be in
`agentTrustDomains`.

##### Managed cluster setup

**1. The platform's SPIRE entry** is on the management cluster, for the one agent, selecting on
the Klusterlet's namespace:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster5-klusterlet-agent
spec:
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster5/agent/klusterlet"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: cluster5
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: klusterlet
  workloadSelectorTemplates:
  - "k8s:sa:cluster5-work-sa"
```

**2. The `Klusterlet`** is the same as for `Hosted`, named `cluster5`, with
`deployOption.mode: SingletonHosted`. The `external-managed-kubeconfig` is handled as in `Hosted`.

**3. The klusterlet operator** installs the same admission policy on the management cluster. Its
only allowed pair is `klusterlet` with `cluster5-work-sa`.

**4. It deploys one agent** in namespace `cluster5` on mgmt1:

| Deployment | ServiceAccount | `spiffe-role` | SVID for cluster5 |
| --- | --- | --- | --- |
| `cluster5-agent` | `cluster5-work-sa` | `klusterlet` | `spiffe://east.example.org/ocm/cluster/cluster5/agent/klusterlet` |

##### Join

```shell
clusteradm join --cluster-name cluster5 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server \
  --spiffe-trust-domain east.example.org \
  --mode hosted --singleton --managed-cluster-kubeconfig cluster5.kubeconfig
clusteradm accept --clusters cluster5 --spiffe-trust-domain east.example.org
```

The steps are the same as in [Singleton](#join-1): one agent, mapped to
`system:open-cluster-management:cluster5:klusterlet`, doing both registration and work. As in
`Hosted`, cluster5 is pinned to the management cluster's trust domain. For its add-on agents, see
[Add-on agents](#add-on-agents).

##### Renew

The same as in [Default](#renew), with one SVID, rotated by the management cluster's SPIRE.

##### Revoke

The same as in [Hosted](#revoke-2): the platform removes the `ocm-cluster5-klusterlet-agent` entry
on the management cluster before deleting the `ManagedCluster`.

#### Add-on agents

Add-ons follow the same model as the Klusterlet agents. Each add-on agent has its own SVID; the
registration agent no longer obtains credentials on the add-on's behalf. This part goes through the
same steps as the install modes: hub setup, managed cluster setup, join, renew and revoke. The
example is `my-addon` on cluster2, which is pinned to `east.example.org`.

##### Hub setup

Nothing new is deployed on the hub. Add-on agents use the same SPIFFE listener as the Klusterlet
agents, and the add-on is defined on the hub as today. What changes is one check in the add-on
manager (`pkg/addon` in ocm).

**The add-on manager checks that an add-on's `kubeClient` subject can be carried by an SVID.** The
hub derives an add-on agent's user and groups from its SPIFFE ID, so a subject that does not match
that mapping can never be presented. The comparison uses the full mapped identity, including
`system:authenticated`, so a default subject that lists that group still matches.

| `kubeClient` subject | Result |
| --- | --- |
| The default, `system:open-cluster-management:cluster:cluster1:addon:my-addon:agent:my-addon-agent` | Matches `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/addon/my-addon/agent/my-addon-agent`. |
| A custom user such as `my-team-bot` | No SVID can carry it, so the add-on gets a `RegistrationApplied=False` condition. |

**In short:** the hub needs no new component for add-ons, only a check that each add-on's subject
is one an SVID can carry.

##### Managed cluster setup

**1. The admin switches the cluster's add-ons to SPIFFE** with the existing
`addOnKubeClientRegistrationDriver` field of the `Klusterlet`. It is required when the Klusterlet
itself uses `spiffe`:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster2
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      # ... as in "Managed cluster setup" for the install mode
    addOnKubeClientRegistrationDriver:
      authType: spiffe       # required when registrationDriver.authType is spiffe
```

**2. The platform gives each add-on agent its identity.** As for the Klusterlet agents, OCM does
not create the entry. The add-on's entry selects on its namespace, its ServiceAccount and
`spiffe-role: addon`. For `my-addon` on cluster2, installed in
`open-cluster-management-agent-addon` and running as `my-addon-agent-sa`:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-my-addon-agent
spec:
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster2/addon/my-addon/agent/my-addon-agent"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent-addon
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: addon
  workloadSelectorTemplates:
  - "k8s:sa:my-addon-agent-sa"
```

**3. The add-on framework renders the add-on's workloads with the Workload API.** When a cluster
uses the `spiffe` add-on driver, add-on Deployments, DaemonSets and StatefulSets rendered by the
framework get the `csi.spiffe.io` volume and the `spiffe-role: addon` label instead of the
`<addon>-hub-kubeconfig` Secret volume. Unit tests cover all three kinds.

```yaml
spec:
  template:
    metadata:
      labels:
        open-cluster-management.io/spiffe-role: addon
    spec:
      serviceAccountName: my-addon-agent-sa
      volumes:
      - name: spiffe-workload-api      # replaces the my-addon-hub-kubeconfig Secret volume
        csi:
          driver: csi.spiffe.io
          readOnly: true
```

The framework also gets a helper that builds an add-on agent's gRPC clients from its SVID, so
add-on authors do not have to handle SVIDs themselves.

**4. The Klusterlet operator protects the add-on namespaces.** Add-on agent namespaces get the
same admission policy as the agent namespaces (see [Admission policy](#admission-policy)), with
three differences:

1. **The work agent is the allowed creator.** The allowed creator of Deployments is the work agent,
   which applies the add-on's manifests from the hub. So every add-on pod traces back to a manifest
   the hub defined.
2. **DaemonSets and StatefulSets are allowed.** Many add-ons run one pod per node or need stable
   storage. The same chain applies: only the work agent may create or change them, and a Pod is
   admitted only when the DaemonSet or StatefulSet controller created it for one the work agent
   created. Jobs and CronJobs stay refused: a Job's pods are short-lived and a CronJob keeps
   creating new ones, which makes the chain harder to check, and add-on agents are long-running.
3. **Add-on roles instead of the pairs table.** Add-on pods use their own ServiceAccounts and the
   role label value `addon`. The policy refuses, in an add-on namespace, any pod with a Klusterlet
   ServiceAccount or with the role `registration`, `work` or `klusterlet`.

| Pod in `my-addon`'s namespace | Result |
| --- | --- |
| Runs as `my-addon-agent-sa` with `spiffe-role: addon`, from a Deployment the work agent applied | Admitted |
| Runs as `my-addon-agent-sa` with `spiffe-role: addon`, from a DaemonSet or Deployment another user created | Refused |
| Runs as `klusterlet-work-sa` | Refused |

If the policy is removed, the operator's recovery also cleans the add-on namespaces, and the work
agent re-applies the add-on workloads (see [If the policy is removed](#if-the-policy-is-removed)).

**In short:** each add-on agent mounts the Workload API and has no Secret, and only pods from
manifests the hub defined can get its SVID.

##### Join

```shell
clusteradm addon enable --names my-addon --clusters cluster2
```

What happens when `my-addon` is enabled on cluster2:

1. **Hub.** The add-on is enabled as today, by creating the `ManagedClusterAddOn`. The add-on
   manager sets the add-on's `kubeClient` registration and creates its hub RBAC from the add-on's
   permission configuration, bound to the add-on's user and groups.
2. **cluster2, registration agent** (`pkg/registration/spoke/addon` in ocm). With the `spiffe`
   add-on driver, it no longer creates a CSR, token or `<addon>-hub-kubeconfig` Secret for
   `kubeClient` registrations. It sets the new `ManagedClusterAddOn` (`v1beta1`) value `spiffe` in
   `status.registrations[].kubeClient.driver`, so the hub can see how the add-on authenticates, and
   the `RegistrationApplied` condition. No new registration type is needed:

   ```yaml
   apiVersion: addon.open-cluster-management.io/v1beta1
   kind: ManagedClusterAddOn
   metadata:
     name: my-addon
     namespace: cluster2
   status:
     registrations:
     - type: kubeClient
       kubeClient:
         driver: spiffe
     conditions:
     - type: RegistrationApplied
       status: "True"
   ```

   A `customSigner` registration is refused with a clear condition, because the hub does not sign
   certificates in this mode.
3. **cluster2, work agent.** It applies the add-on's manifests from the hub, so the add-on agent
   starts with the CSI volume and the `spiffe-role: addon` label.
4. **cluster2, add-on agent.** It gets its own SVID from east's SPIRE,
   `spiffe://east.example.org/ocm/cluster/cluster2/addon/my-addon/agent/my-addon-agent`, picked by
   ID like the Klusterlet agents' SVIDs, and connects to the hub over mTLS, accepting only the hub's
   ID. It must be in `east.example.org`, because cluster2 is pinned there (see
   [Trust domain pinning](#trust-domain-pinning)).
5. **Hub.** It maps the SVID to the user
   `system:open-cluster-management:cluster:cluster2:addon:my-addon:agent:my-addon-agent` and the
   groups `system:open-cluster-management:cluster:cluster2:addon:my-addon`,
   `system:open-cluster-management:addon:my-addon` and `system:authenticated`. These are the
   default `kubeClient` subject, so the add-on's existing hub RBAC applies. The hub checks both the
   cluster and the add-on in the SVID against every request, so the agent of `my-addon` cannot
   touch `other-addon`, or another cluster's `my-addon`.

**In short:** the add-on agent registers with its own SVID; the registration agent only reports
that the add-on uses `spiffe`.

##### Renew

Exactly as for the Klusterlet agents (see [Renew](#renew)): SPIRE rotates the add-on agent's SVID
and pushes it over the Workload API, and the hub closes the add-on agent's connection when the SVID
it was opened with expires. The agent reconnects with its current SVID.

##### Revoke

| The admin does this on the hub | What happens |
| --- | --- |
| Disables the add-on, by deleting the `ManagedClusterAddOn` `my-addon` in namespace `cluster2` | The add-on manager removes its RBAC, and the hub gRPC server closes every connection of `my-addon`'s agents on `cluster2`. |
| Sets `hubAcceptsClient: false` on `cluster2`, or deletes `cluster2` | The hub closes the connections of all cluster2's add-on agents, along with the Klusterlet agents'. |

In SPIRE, deleting the add-on's registration entries stops new SVIDs; existing ones lapse within
their TTL.

##### In each install mode

Add-on agents run where the add-on is installed and get their SVIDs from the local SPIRE agent
there. Like every agent of a cluster, they must be in the trust domain the cluster is pinned to.

| Mode | Where add-on agents run | Which trust domain their SVIDs must be in |
| --- | --- | --- |
| `Default`, `Singleton` | On the managed cluster | The managed cluster's, which the cluster is pinned to. |
| `Hosted`, `SingletonHosted` | On the managed cluster, or on the management cluster for hosted add-ons | The management cluster's, which the cluster is pinned to. A hosted add-on on the management cluster is in it. An add-on agent on the managed cluster itself is in it only if the managed cluster's SPIRE is also in that trust domain; otherwise the hub refuses it. |

Example: cluster4 is hosted on mgmt1 and pinned to `east.example.org`. A hosted `my-addon` agent
on mgmt1 gets `spiffe://east.example.org/ocm/cluster/cluster4/addon/my-addon/agent/my-addon-agent`
and is accepted. A `my-addon` agent on cluster4 itself, whose SPIRE is in `west.example.org`,
would present `spiffe://west.example.org/...` and is refused.

#### Changes by repository

| Repository | Package | Change |
| --- | --- | --- |
| api | operator, addon | `spiffe` driver on `ClusterManager` and `Klusterlet` (fields above), `spiffe` value for `status.registrations[].kubeClient.driver` on `ManagedClusterAddOn` (`v1beta1`), `SPIFFERegistration` feature gate. |
| sdk-go | `pkg/server/grpc/authn` | `SPIFFEAuthenticator`: SPIFFE ID mapping, trust domain and expiry. |
| sdk-go | `pkg/server/grpc` | SVID picker, server TLS from the Workload API, the `TransportCredentials` wrapper, connection registry and expiry timers, interceptors. |
| sdk-go | `pkg/cloudevents/generic/options/grpc` | SVID picker and client TLS that accepts only the hub ID. |
| sdk-go | (module) | New dependency, `github.com/spiffe/go-spiffe/v2`. |
| ocm | `pkg/server/grpc` | The SPIFFE listener and Service, revocation watches. |
| ocm | `pkg/server/services/cluster` | Trust domain annotation set on create and kept on agent updates. |
| ocm | `pkg/registration/hub` | The `spiffe-revocation` finalizer and delete guard. |
| ocm | `pkg/registration/register/spiffe` | The new agent registration driver. |
| ocm | `pkg/registration/spoke/addon` | The `spiffe` add-on driver: no CSR, token or Secret; `customSigner` refused. |
| ocm | `pkg/addon` | The add-on manager's check that a `kubeClient` subject can be carried by an SVID. |
| ocm | `pkg/operator` | SPIFFE versions of the gRPC server and agent Deployments for all four install modes, the admission policy, its checks and recovery. |
| addon-framework | | The client helper, and rendering of add-on Deployments, DaemonSets and StatefulSets with the CSI volume and `spiffe-role: addon` label. Unit tests cover all three kinds. |
| clusteradm | `init`, `join`, `accept` | `--registration-drivers spiffe`, `--spiffe-trust-domain`, `--spiffe-agent-trust-domains`, `--registration-auth spiffe`, `--hub-grpc-endpoint`, `--hub-spiffe-id`. The install modes use the existing `--mode` and `--singleton` flags. |

### Risks and Mitigation

**SPIRE gives a cluster's ID to the wrong workload.**
If SPIRE issues `.../ocm/cluster/cluster1/...` to a pod that is not one of cluster1's agents,
that pod can act as cluster1. Example: a careless registration entry that matches any pod with the
`work` label on any cluster.
*Mitigation:* the platform must tie each cluster's entries to that cluster's SPIRE agents, and select
on namespace, ServiceAccount and role label (see
[Admission policy](#admission-policy)). The admission policy the Klusterlet
operator installs makes sure only operator-owned Deployments can start pods with that combination,
so a user who can create a Job or Deployment in the agent namespace cannot borrow the agent's
identity. The hub also checks that every request is about the cluster named in the SVID, so an
agent of cluster2 can never change cluster1's resources, even with a valid SVID. This is
documented as a deployment requirement.

**One trust domain acts for a cluster in another.**
Example: `cluster2` is pinned to `east.example.org`, and the SPIRE servers of `west.example.org`
issue `spiffe://west.example.org/ocm/cluster/cluster2/agent/work`, by mistake or because they are
compromised.
*Mitigation:* the hub pins each cluster to the trust domain it registered from, the admin confirms
it at acceptance, and the hub refuses requests for `cluster2` from any other trust domain (see
[Trust domain pinning](#trust-domain-pinning)). A compromised trust domain can act only for the clusters pinned
to it, and can only create new, unaccepted clusters. The hub accepts only the trust domains in
`agentTrustDomains`, however many its SPIRE federates with.

**A deleted cluster comes back.**
Example: an admin deletes `ManagedCluster/c1`, but its agent still has an SVID valid for 40
minutes. It reconnects and creates `c1` again.
*Mitigation:* the `spiffe-revocation` finalizer keeps `c1` terminating, so it cannot be recreated
and all its requests are denied, until every SVID issued before the deletion has expired (see
[Deleting a cluster](#deleting-a-cluster)). To keep the cluster out for good, the platform removes
its SPIRE registration entries, and the `ClusterSPIFFEID` objects or other source that creates
them, before deleting the `ManagedCluster`.

**An agent presents the wrong SVID.**
Example: the platform gives every pod a generic SVID as well, and the Workload API lists it first.
The agent would present that ID and the hub would reject it.
*Mitigation:* every OCM component picks its SVID by the SPIFFE ID it expects, never by position,
and reports not ready if that ID is missing.

**In hosted modes, one management cluster runs agents for many clusters.**
Example: a management cluster runs the Klusterlets for `cluster1` and `cluster2` in namespaces
`cluster1` and `cluster2`. A loose entry could give a pod in `cluster2` an ID for `cluster1`.
*Mitigation:* entries on the management cluster select on the Klusterlet namespace and the role
label. A fully compromised management cluster can act for the clusters it hosts, but that is
already true today, because it holds their hub credentials.

**An add-on agent uses an ID for a different add-on or cluster.**
Example: the agent of `my-addon` on cluster1 tries to update `other-addon`'s status.
*Mitigation:* the hub checks both the cluster and the add-on in the SVID against every request.
Add-on entries select on the add-on's namespace and ServiceAccount.

**SPIRE is down.**
If an agent cannot reach its local Workload API, it cannot get a new SVID.
*Mitigation:* open connections keep working until the current SVID expires. Example: with a 1-hour
TTL rotated every 30 minutes, an agent has about 30 minutes of margin. The Klusterlet or
`ManagedClusterAddOn` shows a clear condition while the Workload API is unreachable. The TTL is a
platform choice: shorter means faster revocation, longer means more time to fix SPIRE.

**A connection outlives its SVID.**
A gRPC connection checks identity only once, when it opens.
*Mitigation:* the hub closes each connection when the client's SVID expires (see the 10:40 / 11:00
example in [Server TLS and SVID expiry](#server-tls-and-svid-expiry)), and the agent reconnects
with its current SVID.

**A load balancer ends TLS in front of the hub.**
Example: an HTTPS load balancer that decrypts traffic and re-encrypts it with its own certificate.
The agent would then see the load balancer's certificate, not the hub's SVID.
*Mitigation:* the gRPC endpoint must use a layer 4 (TCP) load balancer that passes TLS through.
Because agents check the hub's exact SPIFFE ID, a wrong setup fails to connect instead of
silently trusting the load balancer.

**Another workload pretends to be the hub.**
Example: any pod in the trust domain with a valid SVID listens on the hub's address.
*Mitigation:* agents accept only `spiffe://ocm-hub.example.org/ocm/hub/grpc-server`, not any ID
in the trust domain.

**Strangers flood the gRPC server.**
*Mitigation:* a client must complete the TLS handshake with a valid SVID before it can send
anything, so unknown clients are dropped early. Today, by contrast, anyone with the bootstrap
token can start the join flow. Network-level protection is still advisable.

**Clocks differ between clusters.**
Example: a managed cluster's clock is 3 minutes ahead, so a fresh SVID looks "not valid yet" to
the hub.
*Mitigation:* clusters must run NTP, and the expiry check allows a small clock difference,
`spiffe.maxClockSkew` on the `ClusterManager` (default `5m`). The delete guard adds the same
allowance to its wait.

Security review should involve the SIG Architecture maintainers and someone familiar with SPIRE
deployments.

## Design Details

### Open Questions

1. Should the SPIFFE ID path scheme be fixed, or configurable with a template on the
   `ClusterManager`?
2. Should the hub support automatic acceptance for clusters whose SVIDs match a configured
   pattern, or always keep acceptance as an explicit step?
3. Should the `ManagedCluster` be created by the agent on first connection (as proposed), or be
   pre-created by the admin with the agent only allowed to update its status?
4. Which SPIRE topology should `clusteradm` and the e2e tests treat as the reference: one shared
   trust domain with every cluster's SPIRE agents attesting to one SPIRE server, nested SPIRE
   servers, or one federated trust domain per cluster?
5. In `Hosted` and `SingletonHosted` modes, agents reach the managed cluster's kube-apiserver with
   the user-provided `external-managed-kubeconfig` and the operator-derived Secrets. Should this
   proposal also replace that credential, for example by having the managed cluster's
   kube-apiserver accept JWT-SVIDs through structured authentication configuration, or is it a
   separate enhancement?
6. Existing add-ons that use their hub kubeconfig to call arbitrary hub APIs need to move to the
   gRPC services. Which additional gRPC services, if any, are needed for common add-ons?

### Test Plan

- **Unit tests.** SPIFFE ID parsing and mapping for every ID form (valid and invalid paths, wrong
  trust domain, cluster and add-on mismatch), the SPIFFE authenticator, connection closure at SVID
  expiry (including that no further unary or stream RPC succeeds on that connection), connection
  shutdown on cluster deny or delete and on add-on disable, and operator
  rendering of the SPIFFE Deployments for each install mode. Also cover the delete guard holding
  the finalizer for `maxSVIDLifetime` plus `maxClockSkew` after deletion, including a SPIRE clock
  ahead of the hub's and an agent holding a newer SVID with a longer TTL than any SVID the hub has seen.
  For trust domains, cover the pinning annotation being set from the SVID on create, ignored in
  the request and kept on agent updates; refusal of an SVID from another configured trust domain
  for a pinned cluster, for both Klusterlet and add-on agents; refusal of trust domains outside
  `agentTrustDomains`; and a cluster without the annotation with one and with several agent
  trust domains.
- **Admission policy tests.** For each of `Default`, `Singleton`, `Hosted` and `SingletonHosted`,
  check that the allowed role and ServiceAccount pairs are accepted, that every other pair is
  refused (for example `klusterlet` with `<klusterlet>-registration-sa`, or a `work` pod when the
  Klusterlet runs in `Singleton` mode), and that Jobs and Deployments not created by the operator are refused. Also check
  that the operator deploys no agent, and reports `Degraded`, when the ValidatingAdmissionPolicy
  API is not served or the policy is missing. For add-on namespaces, check that an add-on pod
  with its own ServiceAccount and `spiffe-role: addon` is accepted when it comes from a Deployment,
  DaemonSet or StatefulSet created by the work agent, that the same workloads created by anyone
  else are refused, that Jobs and CronJobs are refused, and that a pod with a Klusterlet
  ServiceAccount or role is refused. For recovery, remove the policy, then create a new DaemonSet
  and change the image of a DaemonSet the work agent applied. Check that the operator deletes both
  while every agent, the work agent included, is at zero, and that afterwards only the clean
  copy re-applied from the `ManifestWork` exists. Also check that agents do not start, and the
  Klusterlet reports `Degraded`, when the built-in controllers share one identity. With the
  policy in place, check that changing a ReplicaSet's Pod template, changing a Pod's image and
  adding an ephemeral container are all refused, while a Deployment rollout, ReplicaSet scaling
  and Pod status updates still work. Also check that, with the policy in place, a DaemonSet or Deployment
  created by any other user is refused, and that a ReplicaSet or Pod is refused when the requester
  is not the matching built-in controller or the owner kind is wrong.
- **Integration tests.** envtest plus an in-process fake Workload API that issues SVIDs from a
  test CA with short TTLs. Cover register, accept, rotation and reconnect, deny, delete, add-on
  enable, rotation and disable, and the checks that no Secret or ConfigMap with hub credentials or
  key material is created on the hub, managed or management cluster. The hosted-mode
  managed-cluster kubeconfig Secrets are excluded until Open Question 5 is resolved.
  Also cover `spiffe` and `grpc` listed together: a `grpc` agent and a `spiffe` agent both work,
  and one cluster migrates from one to the other.
- **E2E tests.** kind clusters with SPIRE installed from the SPIFFE Helm charts, run for each of
  `Default`, `Singleton`, `Hosted` and `SingletonHosted`: join, accept, ManifestWork apply and
  status, add-on enable and status sync, SVID rotation under load, and revocation of a cluster
  and of a single add-on. Also a hub with managed clusters in two federated trust domains other
  than its own, including an agent of one trust domain trying to act for a cluster of the other.

### Graduation Criteria

- **Alpha.** Behind the `SPIFFERegistration` feature gate. All four install modes and add-on
  registration supported. Unit and integration tests.
- **Beta.** E2E tests for all four modes and add-ons in CI, `clusteradm` and add-on framework
  support, documented SPIRE deployment requirements, and a security review.
- **GA.** Used in production by at least one adopter, with migration from the `grpc` driver
  documented and tested.

### Upgrade / Downgrade Strategy

- The driver is opt-in. Clusters that do not set `authType: spiffe` are unaffected.
- The hub can enable `spiffe` alongside existing drivers in `registrationDrivers`, so clusters
  move one at a time. The `spiffe` listener runs next to the existing `grpc` listener (see
  [gRPC server listeners](#grpc-server-listeners)), so agents that have not migrated keep their
  current endpoint and credentials.
- Because the OCM user and group names are the same as with the `csr` and `grpc` drivers and the
  add-on framework defaults, an existing cluster keeps its ManagedCluster, namespace,
  ManifestWorks, add-ons and RBAC when its Klusterlet switches to `spiffe`. The operator deletes
  the old `hub-kubeconfig-secret` and add-on credential Secrets after the switch.
- An existing `ManagedCluster` has no trust domain annotation. With one agent trust domain it is
  pinned to it automatically; with several, a hub admin sets
  `open-cluster-management.io/spiffe-trust-domain` before the cluster switches.
- Switching between install modes works as today; only where SVIDs are issued changes.
- Downgrade: switch the Klusterlet back to its previous drivers and provide a bootstrap
  kubeconfig, as for a new join.

### Version Skew Strategy

- An agent configured for `spiffe` requires a hub that serves the `spiffe` driver. Against an
  older hub the TLS handshake fails and the Klusterlet reports a degraded condition.
- Older agents keep using their existing drivers against a newer hub.
- An add-on agent built without SPIFFE support cannot run on a cluster that uses the `spiffe`
  add-on driver; the add-on manager reports this on the `ManagedClusterAddOn`.
- The CloudEvents data types and services are unchanged, so add-on and work payloads are
  compatible across versions.

## Implementation History

- 2026-11-03: Proposal created.
- 2026-11-03: Added support for all four install modes and for add-ons.

## Drawbacks

- Requires a SPIRE deployment, with SPIRE agents on the hub, every managed cluster and, in hosted
  modes, every management cluster, which is significant operational work for users who do not
  already run it.
- The security of cluster and add-on identity moves to SPIRE configuration (registration entries
  and node attestation), which OCM cannot validate.
- Add-ons that need direct hub kube-apiserver access, or hub-signed certificates, must be changed
  to use the gRPC services and their SVID.
- In hosted modes, the managed-cluster kubeconfig remains until Open Question 5 is resolved.

## Alternatives

- **Keep the `grpc` driver and back its CA with a KMS.** Removes the signing key from the hub but
  keeps the bootstrap token, CSR flow and per-cluster and per-add-on key Secrets, and adds a
  dependency on an external key service.
- **JWT-SVIDs as bearer tokens.** Works through TLS-terminating proxies, but needs audience
  handling and replay protection, and the hub would have to fetch JWT bundles. X.509-SVID mTLS
  gives proof of possession with no extra moving parts; JWT-SVID support could be added later.
- **SPIRE's Kubernetes CredentialComposer to authenticate agents to the hub kube-apiserver.**
  Keeps agents talking to the kube-apiserver, which gRPC registration exists to avoid, and needs
  apiserver authentication changes that managed Kubernetes services often do not allow.
- **OCM registers SPIRE entries itself through the SPIRE server API.** Would automate entry
  creation, but makes OCM depend on a second external API and gives the hub the power to mint
  any identity.

## Infrastructure Needed

- CI jobs that install SPIRE on kind clusters for the e2e tests, including a management cluster
  for the hosted modes.
