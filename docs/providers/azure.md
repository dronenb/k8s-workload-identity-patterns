# Azure and Microsoft Entra workload identity

Microsoft Entra Workload Identity Federation lets an application registration or
user-assigned managed identity trust a token from an external OIDC issuer. A
Federated Identity Credential (FIC) matches the token's issuer, subject, and
audience and allows the workload to request a Microsoft Entra access token
without a client secret or application private key.

This guide applies the models in [Kubernetes workload identity](../workload-identity.md)
and [multi-tenant identity](../multi-tenant-identity.md).

## Identity choices

| Choice | Result |
| --- | --- |
| Application registration with FIC | A Kubernetes identity authenticates as an Entra application. |
| User-assigned managed identity with FIC | A Kubernetes identity authenticates as the managed identity. |
| Client secret or application certificate | The workload authenticates with long-lived credential material. Not recommended. |

A FIC is trust configuration, not a stored workload credential. Its issuer,
subject, and audience values must case-sensitively match the external assertion.

## Vanilla Kubernetes federation

Any suitable Kubernetes cluster can federate to Microsoft Entra ID. For the
standard FIC flow, publish publicly reachable HTTPS discovery and JWKS endpoints
and use an RS256-signed ServiceAccount token, which is the signing algorithm
Microsoft documents as supported for this flow:

1. Publish a stable OIDC issuer and JWKS endpoint.
2. Create or select an Entra application or user-assigned managed identity.
3. Add a FIC for the Kubernetes issuer, ServiceAccount subject, and expected audience.
4. Project a Kubernetes token for that audience.
5. Send the token as a federated client assertion to the Entra token endpoint.
6. Use the returned access token for the requested Entra-protected resource.

The conventional audience for Azure Workload Identity is
`api://AzureADTokenExchange`.

```text
Kubernetes ServiceAccount JWT
  -> Entra validates FIC and issuer JWKS
  -> Entra access token
  -> Azure, Microsoft Graph, or another Entra-protected API
```

The Entra application or managed identity must separately receive authorization
for the target resource. A successful FIC exchange does not itself grant Azure
RBAC or API permissions.

## Managed ambient identity on AKS

Azure Workload Identity uses a mutating admission webhook to inject a projected
token file and standard environment variables. A typical Pod opts in with the
following label; for a Deployment, put it under
`spec.template.metadata.labels`:

```yaml
metadata:
  labels:
    azure.workload.identity/use: "true"
```

The ServiceAccount selects the target identity with:

```yaml
metadata:
  annotations:
    azure.workload.identity/client-id: 00000000-0000-0000-0000-000000000000
```

It can override the webhook's default tenant with
`azure.workload.identity/tenant-id`.

The injected settings include `AZURE_AUTHORITY_HOST`, `AZURE_CLIENT_ID`,
`AZURE_TENANT_ID`, and `AZURE_FEDERATED_TOKEN_FILE`. The authority host must
match the Azure cloud environment. Supported Azure SDK chains can consume these
inputs without a client secret.

This ambient path is appropriate for an ordinary workload acting as itself and
for a controller acting as its installation identity. It must not select target
namespace identity for a multi-tenant reconcile.

## Explicit federation

The same token-file convention can be configured explicitly on AKS, GKE, EKS,
or another Kubernetes cluster. A workload acting as itself can use an Azure
default chain when the environment contains only the intended workload identity
inputs.

A multi-tenant controller holding delegated tokens in memory should construct a
`ClientAssertionCredential` or language-equivalent credential with a callback
that requests a fresh target assertion when needed. A
`WorkloadIdentityCredential` is suitable when it is configured with a
target-specific token file that is refreshed for the credential's lifetime. It
must not use `DefaultAzureCredential`, because that chain can select the
controller's managed identity, environment credential, CLI identity, or another
installation-level source.

## Controller identity

A controller that needs Azure access for controller-owned operations can use:

- AKS ambient Azure Workload Identity.
- Explicit Azure Workload Identity settings.
- An Azure managed identity available through the hosting environment.
- A client secret or application certificate, which is discouraged.

`DefaultAzureCredential` can be appropriate for these controller-owned calls
when its enabled sources and precedence are understood. That decision has no
bearing on target namespace credential selection.

## Delegated target identity

For `serviceAccountRef`, the controller should:

1. Resolve the ServiceAccount in the target namespace.
2. Configure a renewable assertion callback or target-specific token file for `api://AzureADTokenExchange` or the configured FIC audience.
3. Resolve the target Entra tenant ID, client ID, and authority host.
4. Construct an explicit workload identity or client-assertion credential.
5. Pass that credential to the target Azure SDK client.

Do not populate process-wide `AZURE_*` variables and do not call
`DefaultAzureCredential` for step 4.

The exceptions are an explicitly selected `controllerIdentity` mode or a
controller policy that explicitly allows fallback for the failed credential
stage. In either case, deliberately reuse the already established controller
credential after policy evaluation; do not invoke `DefaultAzureCredential` as
target discovery or implicit fallback.

### Target identity resolution

Resolve the client ID in this order:

1. Explicit Azure client ID in namespace configuration.
2. `azure.workload.identity/client-id` on the target ServiceAccount.
3. Fail if no target client ID is resolved.

Namespace configuration explicitly overrides the annotation. The annotation is
a useful convention even when the delegated controller performs TokenRequest
and the Entra exchange without the AKS webhook.

Resolve the tenant ID in this order:

1. Explicit target tenant in namespace configuration.
2. `azure.workload.identity/tenant-id` on the target ServiceAccount.
3. An explicitly configured installation-wide default for target federation.
4. Fail when none is available.

Do not derive the target tenant from the controller's ambient `AZURE_TENANT_ID`.
Resolve the authority host from explicit provider or installation configuration
for the target Azure cloud, not from whichever controller credential happens to
be selected.

The FIC should restrict the subject to the selected target ServiceAccount. If a
namespace can select arbitrary client IDs, Entra still rejects an assertion
unless the selected identity has a matching FIC.

## Cross-cloud clusters

Microsoft Entra Workload Identity Federation supports external Kubernetes
issuers, including suitable GKE, EKS, and on-premises clusters. The HTTPS issuer
discovery and JWKS endpoints must be publicly reachable by Microsoft Entra ID.
Unlike Google Workload Identity Federation, an Entra FIC has no uploaded-JWKS
alternative for a private issuer.

Cross-cloud workloads normally use explicit federation rather than the AKS
webhook. The underlying assertion and FIC validation are the same.

## Application certificates are not issuer keys

An Entra application certificate credential and a Kubernetes ServiceAccount
issuer key solve different problems.

With a FIC:

```text
Kubernetes signs ServiceAccount JWT
  -> Entra retrieves issuer public keys from JWKS
  -> Entra validates issuer, subject, and audience
```

With an application certificate:

```text
Application possesses certificate private key
  -> application signs private_key_jwt client assertion
  -> Entra verifies the registered public certificate
```

Do not upload the cluster's ServiceAccount signing public key as an application
certificate credential. Its private key represents the entire Kubernetes issuer
and could mint ServiceAccount tokens. It is not an application-held credential,
and a Kubernetes ServiceAccount JWT does not have the same trust registration as
an app certificate assertion.

If application certificate authentication is required, deliver and rotate a
private key dedicated to that application. Treat it as a static target
credential unless a concrete short-lived certificate integration is defined.

## Workload-attested confidential clients

Microsoft Entra ID can accept an external identity-provider JWT as a federated
client assertion for an app registration with a matching FIC. This is useful
beyond Azure resource access, including a backend that must authenticate as a
confidential OAuth client.

For an authorization-code application, keep the identities separate:

```text
User authorization code
  -> represents user authentication

Kubernetes workload assertion
  -> authenticates the confidential backend client
```

Prefer the workload assertion over a client secret. With MSAL, configure the
confidential client with a client-assertion callback, such as
`WithClientAssertion` in MSAL.NET, that supplies a fresh projected workload
token. The callback must remain available for every authorization-code
redemption, token-cache miss, and refresh-token exchange that requires
confidential client authentication. Confirm that the selected language library
and authentication middleware support assertion-based client authentication
for the required flow.

The federated Kubernetes assertion is different from a certificate-signed
assertion whose `iss` and `sub` are the app client ID. Entra recognizes the
external assertion through the FIC's issuer, subject, and audience.

## Static target credentials

A compatibility `secretRef` can contain an Entra client secret or an application
certificate and private key. Construct the target credential directly from that
Secret. Do not inject it into the controller environment where
`EnvironmentCredential` could select it for unrelated reconciles.

This mode is not recommended. Migrate by adding a FIC for the target Kubernetes
ServiceAccount and replacing the Secret reference with `serviceAccountRef`.

## Troubleshooting

Check these boundaries in order:

1. The projected token issuer, subject, and audience exactly match the FIC.
2. Entra can reach the issuer metadata and JWKS endpoint.
3. The target tenant and client ID identify the intended application or managed identity.
4. The Entra principal has Azure RBAC or API authorization for the resource.
5. The target SDK client received an explicit assertion credential, or an explicitly policy-approved controller credential, rather than an implicit `DefaultAzureCredential` fallback.

Do not log the projected token, client assertion, Entra access token, client
secret, or certificate private key.

## Primary references

- [Microsoft Entra Workload Identity Federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [AKS Workload Identity overview](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [Azure Identity WorkloadIdentityCredential](https://learn.microsoft.com/dotnet/api/azure.identity.workloadidentitycredential)
- [Azure Identity ClientAssertionCredential](https://learn.microsoft.com/dotnet/api/azure.identity.clientassertioncredential)
- [Microsoft identity platform authorization code flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-auth-code-flow)
- [MSAL.NET client assertions](https://learn.microsoft.com/entra/msal/dotnet/acquiring-tokens/web-apps-apis/confidential-client-assertions)
