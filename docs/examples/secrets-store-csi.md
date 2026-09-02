# Secrets Store CSI examples

These examples show workload identity as a mounted-file delivery path. Each one
uses the pod's identity, not the controller's, and should be treated as a
workload-scoped credential flow.

The auth shape is provider-specific. There is no shared `parameters.auth` field
across all providers; use the provider's documented knob for the target identity
flow.

## Google Cloud Secret Manager

This follows the known-good demo pattern from
[`kubecon-2024-wif-demo`](https://github.com/dronenb/kubecon-2024-wif-demo):
the CSI provider uses `pod-adc` for the application pod's ServiceAccount.
That means the provider asks the Kubernetes API for a token for the target pod,
then exchanges it for Google credentials. `GOOGLE_APPLICATION_CREDENTIALS` can
still supply external-account metadata in fleet WIF mode, but it is not the
primary auth source. The application pod only consumes the mounted secret
files.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: demo
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-sm
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/PROJECT_ID/secrets/SECRET_NAME/versions/latest"
        path: "secret.txt"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      serviceAccountName: app
      containers:
        - name: app
          image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
          command: ["sh", "-c", "sleep infinity"]
          volumeMounts:
            - name: secrets
              mountPath: /var/secrets
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: gcp-sm
```

Notes:

- On GKE, use Workload Identity and a direct KSA principal or GSA impersonation.
- On Standard clusters, ensure the node pools and metadata-server settings match the GKE Workload Identity setup.
- The pod's identity, not the controller's, authorizes Google Secret Manager access.
- The CSI provider consumes the target pod's ServiceAccount token through the
  pod-adc flow, and may also read `GOOGLE_APPLICATION_CREDENTIALS` for
  external-account metadata.

## AWS Secrets Manager

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-irsa
  namespace: demo
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/demo-secrets-reader
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-sm-irsa
spec:
  provider: aws
  parameters:
    region: us-east-1
    usePodIdentity: "false"
    objects: |
      - objectName: "MySecret"
        objectType: "secretsmanager"
        objectAlias: secret.txt
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-irsa
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-irsa
  template:
    metadata:
      labels:
        app: app-irsa
    spec:
      serviceAccountName: app-irsa
      containers:
        - name: app
          image: public.ecr.aws/docker/library/busybox:1.36
          command: ["sh", "-c", "sleep infinity"]
          volumeMounts:
            - name: secrets
              mountPath: /mnt/secrets-store
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: aws-sm-irsa
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-pid
  namespace: demo
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-sm-pid
spec:
  provider: aws
  parameters:
    region: us-east-1
    usePodIdentity: "true"
    objects: |
      - objectName: "MySecret"
        objectType: "secretsmanager"
        objectAlias: secret.txt
```

Notes:

- IRSA uses the ServiceAccount annotation and a projected web identity token.
- Pod Identity uses an EKS association outside the YAML and should not be mixed with IRSA on the same store.
- Separate the pod's identity from the controller's AWS identity.

## Azure Key Vault

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-azure
  namespace: demo
  annotations:
    azure.workload.identity/client-id: 00000000-0000-0000-0000-000000000000
    azure.workload.identity/tenant-id: 11111111-1111-1111-1111-111111111111
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-wi
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: 00000000-0000-0000-0000-000000000000
    keyvaultName: kvname
    tenantID: 11111111-1111-1111-1111-111111111111
    objects: |
      array:
        - |
          objectName: secret1
          objectType: secret
          objectVersion: ""
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-azure
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-azure
  template:
    metadata:
      labels:
        app: app-azure
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: app-azure
      containers:
        - name: app
          image: mcr.microsoft.com/oss/busybox/busybox:1.36
          command: ["sh", "-c", "sleep infinity"]
          volumeMounts:
            - name: secrets
              mountPath: /mnt/secrets-store
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: azure-kv-wi
```

Notes:

- The `azure.workload.identity/use` label belongs on the pod template.
- The pod's ServiceAccount annotations and CSI `clientID`/`tenantID` should agree.
- Use workload identity; avoid long-lived service principal secrets where possible.

## Static compatibility path

The CSI driver and providers also support legacy, static credential modes.
Those are explicit workload credentials, not controller fallbacks, and should be
treated as migration-only examples.
