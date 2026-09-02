# Kubernetes workload identity

Workload identity lets a Kubernetes workload prove its identity without storing a
long-lived cloud key. A projected Kubernetes ServiceAccount token is commonly
used as a short-lived assertion that an external identity provider validates and
exchanges for an access token.

This document defines the provider-neutral model. See
[multi-tenant identity](multi-tenant-identity.md) for operators that act for
target namespaces and the [provider guides](#provider-guides) for concrete cloud
behavior.

## Identity, provisioning, and consumption

Three separate questions determine the security properties of an integration:

1. Which principal should the operation represent?
2. How is proof of that identity delivered?
3. How does the application select and consume the credential?

### Identity planes

| Identity plane | Meaning |
| --- | --- |
| Workload identity | A workload acts as its own Kubernetes-derived identity. |
| Controller identity | A controller acts as its installation-scoped identity. |
| Target identity | A controller acts for a workload or tenant in a target namespace. |

A controller can need its own cloud identity for installation-owned resources,
shared infrastructure, or telemetry. That does not make the controller identity
appropriate for target namespace operations.

### Credential provisioning

| Method | Description | Guidance |
| --- | --- | --- |
| Managed ambient | The platform provides metadata, token files, environment variables, or credential endpoints. | Prefer when the platform integration matches the target provider. |
| Explicit federation | The workload explicitly projects a token and configures a standard SDK credential. | Prefer for portable and cross-cloud deployments. |
| Brokered identity | An authenticated broker obtains identity for another workload. | Use for specialized delegation; check API maturity. |
| Static credentials | A long-lived key, client secret, or private key is injected. | Avoid and migrate to federation. |

Ambient does not mean unconfigured. ServiceAccount annotations, pod labels,
cloud trust policies, and provider associations still configure the identity.
It means application code can use the environment supplied by the platform.

### Credential consumption

| Context | Credential selection |
| --- | --- |
| A workload acting as itself | A default credential chain is usually appropriate. |
| A controller acting as itself | A default credential chain is usually appropriate. |
| A controller acting for a target namespace | Construct an explicit target credential; do not use the process default chain. |
| An explicitly allowed controller-identity exception | The controller credential may be used because policy deliberately selected or allowed it. |

The same provisioning method can appear in different identity planes. For
example, both a controller and an ordinary workload can use ambient identity,
but they remain different principals with different authorization.

## Kubernetes identity tokens

Modern workload identity starts with a projected, bound ServiceAccount token.
The kube-apiserver TokenRequest API issues a signed JWT with claims that include:

| Claim | Purpose |
| --- | --- |
| `iss` | Identifies the Kubernetes OIDC issuer. |
| `sub` | Identifies the ServiceAccount, normally as `system:serviceaccount:<namespace>:<name>`. |
| `aud` | Limits the intended recipient of the token. |
| `exp` | Limits the token lifetime. |

Projected tokens are short-lived and rotated by Kubernetes. They should replace
legacy auto-generated ServiceAccount token Secrets, which were long-lived and
not audience-bound.

An external provider establishes trust in selected token claims:

```text
Pod or controller
  -> Kubernetes TokenRequest
  -> audience-bound ServiceAccount JWT
  -> external provider validates issuer, signature, subject, audience, expiry
  -> short-lived provider access token or session
```

An audience is part of the trust boundary. Request the audience expected by the
target provider and do not reuse a token issued for one recipient with another.
Provider-side trust should restrict the issuer and subject as narrowly as the
use case permits.

## OIDC discovery and signing keys

An external provider commonly retrieves:

- OIDC metadata from `/.well-known/openid-configuration`.
- Public signing keys from the metadata document's `jwks_uri`.

The issuer URL in a token must match the configured issuer exactly. Issuer and
JWKS endpoints often need to be reachable from the provider's control plane,
not merely from workloads in the cluster.

Private clusters require special consideration. Depending on the provider, an
operator must expose discovery and JWKS endpoints, use a publicly reachable
issuer that represents the cluster, or upload JWKS data where that provider
supports it. Uploaded keys create an operational key-rotation responsibility.

Never upload a Kubernetes ServiceAccount signing key as an application
certificate credential. The signing key represents the Kubernetes issuer; an
application certificate credential represents possession of a private key by
one application. They are different trust models.

## Managed ambient identity

Managed Kubernetes products provide different, non-portable ambient mechanisms:

| Integration | Application-visible mechanism |
| --- | --- |
| GKE Workload Identity Federation | A metadata-compatible server supplies Google credentials. |
| Azure Workload Identity on AKS | A webhook injects a projected token file and `AZURE_*` settings. |
| IAM roles for service accounts (IRSA) | A webhook injects a token file and AWS role settings. |
| EKS Pod Identity | A node agent exposes a container credential endpoint. |

These integrations are convenient when the cluster and target provider match.
They do not define the portable Kubernetes federation model. A GKE workload can
federate to Azure or AWS, an AKS workload can federate to Google or AWS, and an
EKS workload can federate to Google or Azure when the target provider accepts
the cluster issuer and token claims.

## Explicit federation

Explicit federation exposes the token and target configuration that a managed
integration would otherwise inject. Typical standard SDK inputs include:

| Provider | Typical explicit inputs |
| --- | --- |
| Google | An `external_account` configuration and subject-token source, often selected by `GOOGLE_APPLICATION_CREDENTIALS`. |
| Azure | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_FEDERATED_TOKEN_FILE`. |
| AWS | `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. |

These settings can feed a default chain for a workload acting as itself. A
multi-tenant controller must instead pass the target token and target identity
to a provider-specific credential constructor or exchange API. It must not
change process-wide environment variables while reconciling target resources.

Explicit federation remains workload identity. It is not less secure merely
because the deployment owns more configuration than a managed integration.

## Default credential chains

Default credential chains are designed to discover the identity of the running
process. They are valuable for ordinary workloads and for a controller's own
identity, but they are the wrong abstraction for a controller selecting among
many target namespace identities.

Static credentials can also take precedence over federation:

- `GOOGLE_APPLICATION_CREDENTIALS` can identify either an external-account
  configuration or a service account key.
- Azure environment credentials can be attempted before workload identity in a
  chained credential.
- AWS static environment credentials can be selected before web identity.

Do not inject static and federated credential settings into the same workload
unless the precedence is intentional and tested. Enable SDK credential-source
diagnostics when troubleshooting, but never log tokens or private key material.

## Delegated target credentials

When a controller acts for a target namespace, it should obtain an assertion for
the target and construct the credential explicitly:

```text
Target resource
  -> target identity reference
  -> target assertion
  -> explicit provider credential or exchange
  -> target provider principal
```

The implementation must not ask the default chain to infer the target. Failure
to resolve or exchange a target credential fails closed unless controller
policy explicitly permits fallback for that failure stage. A resource
authorization denial after obtaining a target credential should not trigger a
retry with the more privileged controller identity. See
[multi-tenant identity](multi-tenant-identity.md) for the complete model.

## Workload-attested OAuth client authentication

Workload identity is also useful when a backend is a confidential OAuth or
OpenID Connect client. User authentication and client authentication are
separate:

```text
User authorization code
  -> represents the authenticated user

Workload assertion
  -> authenticates the backend redeeming the code
```

When an authorization server supports a federated client assertion, prefer a
short-lived workload assertion over a client secret or a locally distributed
certificate key. Support is provider-specific and must not be inferred from
support for cloud API workload identity.

Microsoft Entra ID supports an external identity-provider JWT as a federated
client assertion for an application with a matching Federated Identity
Credential. Google Sign-In currently documents a client secret for its
server-side code exchange, and Amazon Cognito currently documents
`client_secret_basic` and `client_secret_post` for confidential clients. The
provider guides describe these differences.

Where federated client authentication is unavailable, consider a public client
with Authorization Code with PKCE when that architecture is appropriate. Do not
misrepresent a public client as confidential merely to avoid secret storage.

## Static credentials

Static credentials include:

- Google service account JSON keys.
- Azure client secrets.
- Azure application certificates whose private keys are delivered to a pod.
- AWS access key IDs and secret access keys.
- Legacy non-expiring Kubernetes ServiceAccount tokens.

They are discouraged because they can be copied and replayed, have weak
workload attribution, require storage and rotation, and commonly outlive the pod
that received them. If compatibility requires a static target credential, make
it an explicit target-namespace Secret reference rather than reusing a
controller credential. Keep that path visibly deprecated and provide a
migration to workload federation.

## Related Kubernetes and SPIFFE mechanisms

CSI driver ServiceAccount token requests are stable since Kubernetes 1.22 and
let a driver receive audience-bound tokens through the CSI protocol. Kubernetes
Pod Certificates are stable in Kubernetes 1.37 and provide short-lived,
pod-bound X.509 credentials through a projected volume. Older clusters require
different mechanisms, and clusters must still configure a suitable signer.

The SPIFFE Workload API delivers X.509-SVIDs or JWT-SVIDs to the local workload.
The developing SPIFFE Broker API addresses obtaining identity for another
workload. An SVID is an identity document, not automatically a Google, Azure, or
AWS credential; a compatible cloud trust or exchange is still required.

The SPIFFE Broker API is Incubating rather than Stable. WIT-SVID is also an
Incubating proof-of-possession credential profile; do not treat it as a stable
bearer-token replacement or assume cloud provider support.

## Provider guides

- [Google Cloud](providers/google-cloud.md)
- [Microsoft Azure](providers/azure.md)
- [Amazon Web Services](providers/aws.md)
- [SPIFFE](providers/spiffe.md)

## Primary references

- [Kubernetes ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Kubernetes TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
- [CSI ServiceAccount token requests](https://kubernetes-csi.github.io/docs/token-requests.html)
- [Kubernetes Pod Certificate projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/#podcertificate)
- [SPIFFE specifications](https://github.com/spiffe/spiffe/tree/main/standards)
- [SPIFFE Broker API](https://spiffe.io/docs/latest/spiffe-specs/spiffe_broker_api/)
- [SPIFFE WIT-SVID](https://spiffe.io/docs/latest/spiffe-specs/wit-svid/)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
