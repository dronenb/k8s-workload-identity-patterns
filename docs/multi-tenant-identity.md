# Multi-tenant and delegated identity

A multi-tenant controller reconciles resources for namespaces that may belong to
different teams, security domains, or cloud accounts. Its installation identity
and each target namespace identity are separate security principals.

This distinction applies even when both identities use Kubernetes workload
identity. See [Kubernetes workload identity](workload-identity.md) for the
provider-neutral token and credential model.

## Two identity planes

### Controller identity

The controller identity belongs to the operator installation. It can be used
for controller-owned activities such as shared infrastructure, telemetry, or
control-plane resources that are not attributable to one tenant.

The controller can obtain its own identity through:

- Managed ambient workload identity.
- Explicit workload identity federation.
- Static keys or secrets, which are discouraged.

For these operations, the controller can use the provider's default credential
chain. This is the chain for the controller process, not for target namespaces.

### Target namespace identity

The target identity belongs to the tenant or workload being reconciled. Use it
when authorization, audit attribution, effects, or billing belong to that
tenant.

```text
Controller-owned operation
  -> controller credential source
  -> controller principal

Target operation
  -> target identity reference
  -> explicit target credential
  -> target principal
```

The controller's ambient, explicitly configured, or static credential must not
implicitly become the target credential.

## Target identity sources

A tenant-facing resource or referenced namespace configuration should select
exactly one effective target identity source:

| Source | Purpose | Guidance |
| --- | --- | --- |
| `serviceAccountRef` | Obtain a target Kubernetes JWT and federate it to the cloud provider. | Recommended. |
| SPIFFE Broker reference | Obtain an SVID for the referenced workload. | Incubating; use only with a defined relying-party integration. |
| `secretRef` | Read static target credentials from a Kubernetes Secret. | Not recommended; compatibility only. |
| `controllerIdentity` | Deliberately use the controller principal. | Explicit, policy-controlled exception. |

The following fragments illustrate behavior; they do not define a Kubernetes
API group, version, or kind:

```yaml
identity:
  serviceAccountRef:
    name: tenant-workload
```

```yaml
identity:
  secretRef:
    name: legacy-cloud-credentials
```

```yaml
identity:
  controllerIdentity: {}
```

ServiceAccount and Secret references must resolve within the target resource's
namespace. A namespaced tenant must not use these fields to reference credentials
from another namespace.

Use deterministic configuration precedence:

1. An identity source explicitly selected on the target resource.
2. A complete default identity source from namespace-level configuration.
3. Fail when neither selects a source.

Do not merge partial source definitions from both levels. Namespace policy can
restrict which resource-level sources are permitted. Provider identity fields
are resolved within `serviceAccountRef` as described below rather than being
merged from arbitrary resource fields.

## Delegation with `serviceAccountRef`

`serviceAccountRef` is the preferred source for cloud provider authentication.
The controller requests an audience-bound token for the target ServiceAccount
and explicitly exchanges or consumes that token for the target operation.

```text
Target resource
  -> serviceAccountRef in target namespace
  -> TokenRequest for provider audience
  -> short-lived Kubernetes JWT
  -> explicit provider credential or exchange
  -> target cloud principal
```

The controller should not mount every tenant token into its pod. It should
request a token when needed, bound to the intended audience and with the shortest
practical lifetime.

Permission to create a token for another ServiceAccount is an impersonation
capability. Scope TokenRequest RBAC narrowly and validate the referenced
ServiceAccount in application policy. Do not assume that an RBAC
`resourceNames` restriction fully constrains the `serviceaccounts/token`
subresource without testing it against the Kubernetes versions in use.

## Provider identity resolution

Selecting a Kubernetes ServiceAccount and selecting the external cloud identity
are related but distinct steps. Within `serviceAccountRef` mode, resolve the
provider identity in this order:

1. An explicit provider identity in namespace-level configuration.
2. A standard provider annotation on the referenced ServiceAccount.
3. The provider-specific fallback.

| Provider | ServiceAccount annotation | Fallback |
| --- | --- | --- |
| Google | `iam.gke.io/gcp-service-account` | Direct federated principal access. |
| Azure | `azure.workload.identity/client-id` | Fail if no target client ID is resolved. |
| AWS | `eks.amazonaws.com/role-arn` | Fail if no target role ARN is resolved. |

These annotations are useful identity-selection conventions even when the
controller is not relying on the managed GKE, AKS, or EKS webhook. The
controller can read the annotation and perform the exchange itself.

Namespace-level configuration explicitly overrides the annotation. The API
should preserve three states rather than treating an empty value ambiguously:

- Inherit from the referenced ServiceAccount annotation.
- Select an explicit provider identity.
- Select an explicit direct-federation mode where the provider supports it.

For Google, inherit mode uses the annotation when present and direct federation
when absent. Explicit direct mode ignores a Google service account annotation.
For Azure and AWS, a missing explicit value and missing annotation is an error.

Changing a ServiceAccount annotation or namespace identity configuration can
change the target cloud principal. Treat permission to make those changes as
security-sensitive. Cloud-side subject and authorization rules remain the final
boundary and should reject identities a namespace is not permitted to select.

## Explicit target credential construction

The default credential chain describes the controller process and must not be
used to discover target credentials.

| Provider | Target behavior |
| --- | --- |
| Google | Construct an external-account credential using the target token. Use the federated principal directly or explicitly configure Google service account impersonation. |
| Azure | Construct a workload-identity or client-assertion credential using the target token, tenant, and client ID. Do not use `DefaultAzureCredential`. |
| AWS | Invoke or configure `AssumeRoleWithWebIdentity` with the target token and role ARN. Do not use the process-wide default provider chain. |
| Static Secret | Construct the provider credential directly from the referenced Secret. Do not inject it into the process environment. |

Implementations must not change `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_*`,
`AWS_*`, or similar process-wide settings during reconciliation. Concurrent
reconciles would otherwise race and could issue an operation under another
tenant's identity.

Provider clients that hold credentials must be target-scoped. A cache key should
include all information that can change credential authority:

- Identity plane.
- Namespace and target resource identity.
- ServiceAccount UID, or Secret UID and resource version.
- Provider and audience.
- Resolved cloud identity.
- Direct or impersonation mode.
- Relevant trust configuration.

Expire or refresh cached entries before their underlying token or cloud session
expires. Do not allow a controller client and target client to share a cache
entry merely because they resolve to the same textual cloud identity.

## SPIFFE Broker

The SPIFFE Workload API returns identity for the local workload connected to its
socket. Delegated operators need a different primitive when they request
identity for a referenced target workload. The developing SPIFFE Broker API is
intended for that class of use case.

```text
Controller authenticates to broker
  -> broker authorizes target workload reference
  -> provider resolves reference and verifies existence and SVID entitlement
  -> broker returns target SVID
```

Keep three principals distinct:

- The controller identity used to authenticate to the broker.
- The target workload identity represented by the returned SVID.
- Any cloud identity obtained from a subsequent exchange.

`SPIFFE_ENDPOINT_SOCKET` identifies the local Workload API endpoint.
`SPIFFE_BROKER_SOCKET` is associated with the developing broker model and must
be documented as incubating rather than as a stable cross-platform contract.

An X.509-SVID or JWT-SVID is not automatically a cloud credential. The relying
party must directly trust that SVID or define a provider-supported exchange.
The broker should deliver identity, not become an undocumented cloud credential
broker.

A namespaced broker workload reference should normally resolve in the target
resource's namespace. The Broker API's Kubernetes object reference can express
another namespace in the same control plane; permit that only when both
controller policy and broker authorization explicitly allow it. Cluster-scoped
references require equivalent explicit policy. If the SVID is exchanged for a
cloud credential, construct an explicit target-scoped credential and client. Do
not feed it into the controller default chain. The same controller fallback
policy applies as for `serviceAccountRef`.

## Static target credentials with `secretRef`

`secretRef` is a compatibility path for providers or applications that cannot
yet use federation:

```text
secretRef in target namespace
  -> controller reads target Secret
  -> explicit provider credential object
  -> target principal
```

Examples include a Google service account JSON key, Azure client secret or
application certificate private key, and AWS access key pair.

Requirements for this mode:

- The Secret reference is explicit and same-namespace.
- The Secret is not the controller's installation credential Secret.
- Credential values never appear in logs, events, errors, or resource status.
- The controller does not copy the values into process-wide environment
  variables.
- The provider client is scoped to the target and discarded or refreshed when
  the Secret UID or resource version changes.
- Documentation and status identify the mode as not recommended.
- Operators provide a migration path to `serviceAccountRef`.

A controller configured with a static installation key has a static controller
identity. A target `secretRef` provides a static target identity. These are not
interchangeable.

## Explicit use of controller identity

Some operations intentionally use one shared controller principal. A target can
select this behavior explicitly with `controllerIdentity`, subject to controller
policy.

Examples can include a single shared resource for all namespaces, centralized
billing and authorization, a provider operation with no delegated equivalent,
or a controlled migration from a legacy single-identity operator.

The controller should expose policy that can reject or scope this mode. Status
and audit records should say that controller identity was selected and identify
the resulting cloud principal. Admission policy should be able to prohibit the
mode in tenant namespaces.

## Controller identity fallback

Target credential resolution and exchange fail closed by default. Falling back
to controller identity is permitted only when controller policy explicitly
allows the particular failure stage.

Merely having a valid ambient, explicit, or static controller credential does
not enable fallback. An allowed fallback policy should enumerate eligible
pre-operation credential failures and be scoped by operation, resource type,
namespace, or another clear boundary. Its use must be visible in status and
audit records rather than hidden as an SDK credential-chain decision.

Do not treat a resource authorization denial obtained with a valid target
credential as an ordinary fallback condition. Retrying a denied operation with
a more privileged controller principal defeats tenant authorization. If a
controller supports such legacy behavior at all, it requires a separately
named, high-risk policy rather than the normal credential-fallback setting.

Explicit `controllerIdentity` and fallback are different:

| Behavior | Meaning |
| --- | --- |
| `controllerIdentity` | Target configuration deliberately selects the controller principal. |
| Allowed fallback | An eligible target credential stage failed and explicit controller policy permits use of the controller principal. |
| Default | Return an error without using controller credentials. |

Do not use a broad default credential chain to implement fallback. Resolve the
policy first, then deliberately select the already established controller
credential if policy allows it.

## Status, errors, and auditing

Record enough non-secret context to explain an identity decision:

- Selected identity source.
- Target namespace and resource.
- Referenced ServiceAccount or Secret name and UID.
- Requested audience.
- Resolved provider identity.
- Direct access or impersonation mode.
- Whether controller identity or allowed fallback was used.

Do not record ServiceAccount JWTs, cloud access tokens, static credential values,
client assertions, private keys, or full provider responses that might contain
credentials.

Authentication and authorization failures should identify the failed stage
without changing identity sources implicitly. Useful stages include reference
resolution, TokenRequest authorization, issuer validation, token exchange, role
assumption, service identity impersonation, and resource authorization.

## Security invariants

1. Controller and target identities are separate principals.
2. Controller credential configuration does not implicitly supply target credentials.
3. `serviceAccountRef` is the preferred delegated cloud identity source.
4. Target cloud clients use explicit credentials rather than the controller default chain.
5. Cross-namespace ServiceAccount and Secret references are rejected; broker references require explicit controller and broker authorization outside the target namespace.
6. Identity source changes are authorization-sensitive.
7. A target credential failure does not use controller identity unless controller policy explicitly allows that failure stage.
8. Allowed controller fallback is scoped, deliberate, and auditable.
9. Static target credentials are explicit and not recommended.
10. Credential material never appears in logs, status, or events.
11. Cache entries cannot collide across identity planes or target principals.
12. Provider-side trust and authorization remain the final security boundary.

## Related provider guidance

- [Google Cloud](providers/google-cloud.md)
- [Microsoft Azure](providers/azure.md)
- [Amazon Web Services](providers/aws.md)
- [SPIFFE](providers/spiffe.md)
