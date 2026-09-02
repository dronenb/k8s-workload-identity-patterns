# Kubernetes workload identity patterns

This repository documents common patterns for giving Kubernetes workloads and
platform controllers short-lived external identities without distributing
long-lived cloud keys.

The guidance covers ordinary workloads, controllers with their own cloud
identity, and multi-tenant controllers that act for target namespaces. It uses a
provider-neutral model with concrete Google Cloud, Microsoft Azure, Amazon Web
Services, and SPIFFE guides.

## Key principles

1. Identity selection and credential delivery are separate decisions.
2. A controller identity and a target namespace identity are separate security principals.
3. A workload or controller acting as itself can normally use its provider's default credential chain.
4. A controller acting for a target namespace should construct an explicit target credential instead of using the controller process's default credential chain.
5. Target credential failures do not fall back to controller credentials unless controller policy explicitly allows the particular failure stage.
6. Projected Kubernetes ServiceAccount tokens and short-lived federation are preferred over static keys.
7. Managed GKE, AKS, and EKS integrations are convenient but provider-specific; cross-cloud federation uses the same underlying Kubernetes identity model.

## Identity model

The principal being represented can be:

| Identity | Purpose |
| --- | --- |
| Workload identity | A workload acts as itself. |
| Controller identity | A controller acts as its installation-scoped principal. |
| Target identity | A controller acts for a workload or tenant in a target namespace. |

Credentials can be supplied through managed ambient integration, explicit
federation, a broker, or static secrets. Delegated target identity is not an
alternative to these delivery mechanisms; it identifies whose credential must
be obtained.

## Documentation

- [Kubernetes workload identity](docs/workload-identity.md) defines the
  provider-neutral token, federation, credential-chain, and confidential-client
  model.
- [Multi-tenant and delegated identity](docs/multi-tenant-identity.md) defines
  controller and target identity separation, `serviceAccountRef`, SPIFFE Broker,
  `secretRef`, controller reuse, and fallback policy.
- [Google Cloud](docs/providers/google-cloud.md) covers direct federation,
  service account impersonation, GKE integration, and explicit external-account
  credentials.
- [Microsoft Azure](docs/providers/azure.md) covers Federated Identity
  Credentials, AKS Workload Identity, explicit client assertions, and Entra
  confidential clients.
- [Amazon Web Services](docs/providers/aws.md) covers IAM OIDC trust, IRSA, EKS
  Pod Identity, and explicit `AssumeRoleWithWebIdentity`.
- [SPIFFE](docs/providers/spiffe.md) covers explicit local identity through
  `SPIFFE_ENDPOINT_SOCKET` and Incubating delegated tenant identity through
  `SPIFFE_BROKER_SOCKET`.

## Cross-cloud support

The Kubernetes hosting provider and target cloud provider do not need to match:

| Kubernetes environment | Google Cloud | Microsoft Azure | AWS | SPIFFE |
| --- | --- | --- | --- | --- |
| GKE | GKE integration or explicit federation | Entra FIC | IAM OIDC trust | Workload API or Incubating Broker API |
| AKS | Google Workload Identity Federation | AKS integration or explicit federation | IAM OIDC trust | Workload API or Incubating Broker API |
| EKS | Google Workload Identity Federation | Entra FIC | IRSA, Pod Identity, or explicit federation | Workload API or Incubating Broker API |
| Other Kubernetes | Workload Identity Federation | Entra FIC | IAM OIDC trust | Workload API or Incubating Broker API |

Cloud support depends on a stable issuer, suitable token claims, provider trust
configuration, issuer and JWKS reachability, SDK support, and cloud-side
authorization. SPIFFE additionally requires a trust domain, workload
attestation, endpoint delivery, and relying-party trust.

## Recommended target pattern

For cloud operations attributed to a target namespace:

```text
target resource
  -> serviceAccountRef
  -> Kubernetes TokenRequest
  -> audience-bound ServiceAccount JWT
  -> explicit provider exchange or credential
  -> target cloud principal
```

Static target credentials can be supplied through an explicit same-namespace
`secretRef` for compatibility, but this is not recommended. The controller's
own ambient, explicit, or static identity does not become the target identity
unless controller policy deliberately selects or permits it.
