# Operator identity survey

Many operators solve the same problem in isolation: who should authenticate, for
which target, and how does the target identity differ from the controller's own
identity? The strongest designs make that boundary explicit. The weaker designs
flatten tenant access into a single controller-wide credential.

## Good patterns to cite

- **External Secrets Operator**: this repo's intended model. See
  [multi-tenant identity](multi-tenant-identity.md),
  [workload identity](workload-identity.md), and the provider guides for
  [Google](providers/google-cloud.md), [Azure](providers/azure.md), and
  [AWS](providers/aws.md). The good path is `serviceAccountRef`-driven, target-
  scoped credential construction, with same-namespace static `secretRef` only as
  a compatibility path. See also the concrete examples in
  [External Secrets examples](examples/external-secrets.md).
- **Azure Service Operator**: strong per-resource or per-scope identity. It uses
  explicit resource, namespace, and global credential scopes, and can combine
  Workload Identity with controlled target refs. Good example of separating
  operator identity from resource target identity.
- **Crossplane**: `ProviderConfig` and `providerConfigRef` are explicit per-resource
  identity selectors, with namespaced and cluster-scoped variants. Good example of
  an identity reference that travels with the target resource.
- **cert-manager**: issuer and solver auth can use explicit `serviceAccountRef`
  or Secret refs, which makes the target identity visible in the configuration.
  Good example for per-reconciler delegation, though ambient cluster-wide use is
  still possible.
- **Cluster API Provider Azure**: `AzureClusterIdentity` is a first-class identity
  object with namespace scoping and allowed namespaces. Good example of
  object-scoped identity management.
- **Config Connector**: good when used in namespaced mode with separate
  ServiceAccounts and Workload Identity bindings. It is useful as an example of
  explicit wiring, but shared or cluster-scoped installs flatten the model.

## Common pitfalls

- **External Secrets Operator cluster-wide patterns**: cluster-scoped installs,
  controller-wide ambient identity, and any target flow that silently falls back
  to controller credentials. The repo explicitly calls out the boundary in
  [multi-tenant identity](multi-tenant-identity.md) and says target failures
  must not retry with controller creds unless policy allows that exact stage.
  The same warning applies to the legacy or compatibility examples in
  [External Secrets examples](examples/external-secrets.md) and
  [Secrets Store CSI examples](examples/secrets-store-csi.md).
- **ExternalDNS**: often relies on controller-wide ambient identity or static
  credentials. It is good for DNS automation, but not a strong pattern for
  tenant-scoped identity delegation.
- **AWS Controllers for Kubernetes (ACK)**: controller credentials usually come
  from a single AWS auth chain. It is a useful control-plane pattern, but it does
  not model target namespace identity separately.
- **KEDA**: not a tenant workload identity pattern for this repo's purposes. It
  primarily runs with controller-side authentication, so it does not show how to
  express a separate target namespace principal.
- **Shared ESO or CSI installs without per-namespace refs**: the operator is a
  shared resource, but the identity boundary is flattened unless each namespace or
  store has its own explicit identity ref.
- **Cluster-wide identity objects without namespace constraints**: these are
  operationally convenient but can be too broad for delegated tenants.

## What to learn from them

- Make the target identity object explicit and namespaced when possible.
- Use controller identity only for controller-owned resources.
- Keep fallback policy deliberate and auditable.
- Prefer short-lived assertions and token exchange over static keys.
- Separate the identity used to manage the operator from the identity used to
  access tenant resources.

## What to avoid

- Inferring tenant identity from controller ambient credentials.
- Treating a global credential as if it were tenant-scoped.
- Relying on default credential chains to discover the target.
- Allowing a failed tenant auth path to silently retry with the controller role.
- Using long-lived keys as the normal path when workload identity is available.
