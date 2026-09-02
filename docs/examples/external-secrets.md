# External Secrets examples

These examples show the target-namespace pattern from
[multi-tenant identity](../multi-tenant-identity.md): a controller resolves a
tenant `ServiceAccount` and builds an explicit target credential for that
namespace.

The controller's own ambient identity is only for controller-owned operations.
Do not use it as a hidden fallback for tenant stores.

## Google Cloud Secret Manager

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-gcp
  namespace: tenant-a
---
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: gcp-sm
  namespace: tenant-a
spec:
  provider:
    gcpsm:
      projectID: tenant-secrets-project
      auth:
        workloadIdentityFederation:
          audience: //iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/pool/providers/provider
          serviceAccountRef:
            name: tenant-gcp
            namespace: tenant-a
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: tenant-a
spec:
  secretStoreRef:
    name: gcp-sm
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: app/password
```

Recommended notes:

- Use a tenant-specific `ServiceAccount` and `serviceAccountRef`.
- Prefer direct Google principal access or explicit impersonation only when needed.
- Keep `projectID` explicit when the controller cannot rely on GKE metadata.

## AWS Secrets Manager

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-aws
  namespace: tenant-a
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/tenant-app
---
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-sm
  namespace: tenant-a
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      role: arn:aws:iam::123456789012:role/tenant-app
      auth:
        jwt:
          serviceAccountRef:
            name: tenant-aws
            namespace: tenant-a
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: tenant-a
spec:
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: app/password
```

Recommended notes:

- Use IRSA-style web identity for tenant stores when possible.
- Keep `role` scoped per store or per tenant namespace.
- Do not fall back to the controller's AWS chain if the tenant role is missing.

## Azure Key Vault

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-azure
  namespace: tenant-a
  annotations:
    azure.workload.identity/client-id: 00000000-0000-0000-0000-000000000000
    azure.workload.identity/tenant-id: 11111111-1111-1111-1111-111111111111
---
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-kv
  namespace: tenant-a
spec:
  provider:
    azurekv:
      vaultUrl: https://my-vault.vault.azure.net
      authType: WorkloadIdentity
      serviceAccountRef:
        name: tenant-azure
        namespace: tenant-a
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: tenant-a
spec:
  secretStoreRef:
    name: azure-kv
    kind: SecretStore
  target:
    name: app-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: secret/app-password
```

Recommended notes:

- Put the `azure.workload.identity/use: "true"` label on the pod template only if the webhook is involved.
- Keep `client-id` and `tenant-id` aligned with the selected Entra identity.
- Use a fresh workload assertion or token file; do not use `DefaultAzureCredential` for tenant discovery.

## Static compatibility path

These older ESO auth fields still exist:

- `auth.secretRef` for Google service account JSON keys.
- `auth.secretRef` for AWS access keys.
- `authSecretRef` for Azure service principal secrets or certificates.

Use them only when migration is blocked. They are explicit target credentials,
not controller fallbacks.
