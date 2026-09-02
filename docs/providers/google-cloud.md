# Google Cloud workload identity

Google Cloud Workload Identity Federation accepts an external workload assertion
and exchanges it through the Security Token Service (STS). The resulting
federated principal can access supported resources directly or impersonate a
Google service account.

This guide applies the models in [Kubernetes workload identity](../workload-identity.md)
and [multi-tenant identity](../multi-tenant-identity.md).

## Identity choices

| Choice | Result |
| --- | --- |
| Direct federation | IAM authorizes the federated Kubernetes principal directly. |
| Service account impersonation | The federated principal obtains a short-lived token for a Google service account. |
| Service account key | A workload uses a long-lived JSON private key. Not recommended. |

Direct federation avoids an extra service identity and impersonation hop, but
not every Google API accepts federated principals in the same way. Service
account impersonation can provide compatibility and a stable Google identity.

## Vanilla Kubernetes federation

A non-GKE cluster can federate to Google Cloud when it has a suitable OIDC
issuer. The high-level configuration is:

1. Configure a workload identity pool.
2. Configure an OIDC workload identity provider for the Kubernetes issuer.
3. Map token claims such as the Kubernetes subject to Google attributes.
4. Restrict accepted audiences and attribute conditions.
5. Grant the federated principal direct resource permissions or permission to impersonate a Google service account.
6. Configure an external-account credential to source the projected Kubernetes token.

```text
Kubernetes ServiceAccount JWT
  -> Google STS validates workload identity provider trust
  -> federated access token
  -> direct resource access
     or Google service account impersonation
```

Issuer, subject, and audience restrictions should identify the intended cluster
and ServiceAccount as narrowly as practical. Attribute mappings determine the
principal and principal-set identifiers used in IAM policies.

## Managed ambient identity on GKE

GKE Workload Identity Federation for GKE provides a metadata-compatible server.
Google client libraries using Application Default Credentials (ADC) can obtain
the pod's configured identity without a mounted service account key. It is
always enabled on Autopilot. Standard clusters must enable it on the cluster and
the relevant node pools; workloads scheduled across mixed node pools may also
need the documented metadata-server node selector.

Depending on configuration, a Kubernetes ServiceAccount can access resources as
a federated principal or use a linked Google service account. The annotation
commonly used to identify the latter is:

```yaml
metadata:
  annotations:
    iam.gke.io/gcp-service-account: workload@project-id.iam.gserviceaccount.com
```

The annotation does not authorize impersonation by itself. The linked Google
service account also needs an IAM binding that permits the Kubernetes
ServiceAccount to impersonate it, normally with
`roles/iam.workloadIdentityUser`.

ADC through the GKE metadata server is appropriate for an ordinary workload
acting as itself and for a controller acting as its installation identity. It
must not be used to infer a delegated target namespace identity.

## Explicit federation

Outside the managed GKE path, Google client libraries can consume an
`external_account` credential configuration. That configuration identifies:

- The workload identity provider resource used as the STS audience.
- The external subject-token type.
- The source of the projected Kubernetes token.
- An optional Google service account impersonation URL.

An ordinary workload can select this configuration with
`GOOGLE_APPLICATION_CREDENTIALS` and use ADC. Ensure that the file is an
external-account configuration rather than a service account key.

For a multi-tenant controller, do not set or change
`GOOGLE_APPLICATION_CREDENTIALS` per reconciliation. Construct a Google
external-account credential or subject-token supplier for the selected target
ServiceAccount and pass that credential to the resource client. The supplier
must request or read a fresh Kubernetes assertion whenever the external-account
credential refreshes; do not close over one immutable TokenRequest JWT for the
lifetime of a cached client.

## Controller identity

A controller that needs Google access for controller-owned operations can use:

- GKE ambient Workload Identity Federation.
- An explicit external-account ADC configuration.
- A service account key, which is discouraged.

That credential represents the controller. It does not become a target
namespace credential merely because it is discoverable through ADC.

## Delegated target identity

For `serviceAccountRef`, the controller should:

1. Resolve the ServiceAccount in the target namespace.
2. Configure a renewable token supplier that requests tokens with the audience expected by the Google workload identity provider.
3. Resolve direct access or Google service account impersonation.
4. Construct an explicit external-account credential using that token.
5. Pass the explicit credential to the target Google API client.

Do not invoke ADC for step 4. ADC describes the controller process and can
select its ambient or static installation identity.

The exceptions are an explicitly selected `controllerIdentity` mode or a
controller policy that explicitly allows fallback for the failed credential
stage. In either case, deliberately reuse the already established controller
credential after policy evaluation; do not invoke ADC as target discovery or
implicit fallback.

### Target identity resolution

Use this order:

1. Explicit Google identity mode in namespace configuration.
2. `iam.gke.io/gcp-service-account` on the target ServiceAccount.
3. Direct federated principal access.

Recommended states are:

| State | Behavior |
| --- | --- |
| Inherit | Impersonate the annotated Google service account, or use direct access if the annotation is absent. |
| Impersonate | Use the explicitly configured Google service account. |
| Direct | Ignore the annotation and authorize the federated principal directly. |

The annotation originated with GKE integration but is also a useful convention
for a delegated controller performing the exchange itself.

### Direct resource access

Direct access means the STS-derived federated principal is granted IAM access to
the target resource. It does not mean that token exchange is skipped. The
external-account credential normally performs the STS exchange internally.

Use direct access when the target API accepts the federated principal and its
IAM principal identifier can be granted the required role. Avoid adding service
account impersonation solely as a default.

### Service account impersonation

When impersonation is selected, grant the federated principal only the required
permission to impersonate the selected Google service account, commonly through
`roles/iam.workloadIdentityUser`. Grant resource permissions to the Google
service account rather than broadly to the controller.

The impersonated credential is still short-lived; no service account private
key is required.

## Cross-cloud clusters

The same Google workload identity provider can trust a suitable issuer from
AKS, EKS, another Kubernetes distribution, or an on-premises cluster. The
cluster's hosting provider does not need to match Google Cloud.

Explicit federation is normally used because the GKE metadata server is not
present. The target controller or workload projects a Kubernetes token and
constructs a Google external-account credential.

## Private issuers and JWKS

Google Workload Identity Federation supports supplying JWKS data when the issuer
keys cannot be obtained in the usual way. This can enable private-cluster
patterns, but the operator becomes responsible for uploading new public signing
keys before the cluster starts using them and removing retired keys safely.
Google currently accepts at most eight uploaded keys, and an update replaces
the uploaded set rather than merging it. Rotate with an overlap that includes
the active and next public keys in the replacement set.

Uploaded JWKS public keys are workload identity provider trust material. They
are not service account keys and do not grant possession of a Google identity by
themselves.

## Static target credentials

A compatibility `secretRef` can contain a Google service account JSON key. The
controller should parse it into a target-scoped credential object rather than
putting it in `GOOGLE_APPLICATION_CREDENTIALS`.

This mode is not recommended because the private key is replayable, typically
outlives a pod, requires rotation, and weakens Kubernetes workload attribution.
Migrate by creating workload identity provider trust and replacing the Secret
reference with `serviceAccountRef`.

## Google user authentication and authorization

Modern Sign in with Google can post an ID token to a backend, which verifies the
token and does not authenticate to Google with a confidential client secret for
that verification step. Google Account OAuth authorization is a separate flow
that can return an authorization code for server-side redemption.

Google's OIDC metadata currently advertises `client_secret_basic` and
`client_secret_post`, not `private_key_jwt`, for token endpoint client
authentication. Google does not document Kubernetes workload identity or a
Google external-account credential as OAuth client authentication. Follow the
current Google Identity Services flow and client-type requirements rather than
assuming Google Cloud workload federation replaces an OAuth client secret.

A backend can use Google user authentication while separately using workload
identity for its own Google Cloud API calls.

## Troubleshooting

Check these boundaries in order:

1. The projected token has the expected issuer, subject, audience, and expiry.
2. Google can validate the configured issuer keys or uploaded JWKS.
3. Attribute mapping and conditions produce the expected federated principal.
4. The principal can access the resource directly or impersonate the selected service account.
5. The resource client received the explicit target credential, or an explicitly policy-approved controller credential, rather than an implicit ADC fallback.

Credential diagnostics may identify the selected source, but logs must not
contain the subject token, STS response, access token, service account key, or
client assertion.

## Primary references

- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Workload Identity Federation with Kubernetes](https://cloud.google.com/iam/docs/workload-identity-federation-with-kubernetes)
- [Workload Identity Federation for GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [External account credential configuration](https://cloud.google.com/iam/docs/workload-identity-federation-with-other-providers#create_a_credential_configuration)
- [Sign in with Google using OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect)
