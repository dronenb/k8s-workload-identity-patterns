# AWS workload identity

AWS IAM can trust a Kubernetes OIDC issuer and allow selected ServiceAccounts to
call AWS Security Token Service (STS) `AssumeRoleWithWebIdentity`. STS returns a
short-lived role session without requiring an AWS access key in the pod.

This guide applies the models in [Kubernetes workload identity](../workload-identity.md)
and [multi-tenant identity](../multi-tenant-identity.md).

## Identity choices

| Choice | Result |
| --- | --- |
| Web identity role assumption | A Kubernetes assertion creates a short-lived IAM role session. |
| EKS Pod Identity | An EKS association and node agent provide role credentials to a pod. |
| Static access key | A workload uses a long-lived access key ID and secret. Not recommended. |

IAM roles for service accounts (IRSA) and EKS Pod Identity are different
mechanisms. IRSA uses a projected OIDC token and `AssumeRoleWithWebIdentity`.
EKS Pod Identity uses an EKS association, a node agent, and a container
credential endpoint.

## Vanilla Kubernetes federation

Any suitable Kubernetes issuer can be registered as an IAM OIDC identity
provider:

1. Publish stable OIDC discovery and JWKS endpoints.
2. Create an IAM OIDC provider for the issuer.
3. Create a role with a web-identity trust policy.
4. Restrict the trust policy by issuer claims such as `sub` and `aud`.
5. Project a Kubernetes token for the trusted audience.
6. Call `AssumeRoleWithWebIdentity` with the token and role ARN.

The conventional audience for IRSA is `sts.amazonaws.com`.

```text
Kubernetes ServiceAccount JWT
  -> STS validates IAM OIDC provider and role trust policy
  -> short-lived IAM role session
  -> AWS resource API
```

`AssumeRoleWithWebIdentity` does not require existing AWS credentials. The trust
policy, rather than the caller's controller role, authorizes the exchange.

## IRSA on EKS

IRSA configures an IAM OIDC provider for the EKS cluster and commonly selects a
role with this ServiceAccount annotation:

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/tenant-workload
```

The EKS admission integration injects `AWS_ROLE_ARN` and
`AWS_WEB_IDENTITY_TOKEN_FILE`. Standard AWS SDK credential chains can use those
values for a pod acting as itself. Those settings do not override credential
sources that appear earlier in a language SDK's chain, such as static
environment credentials or an explicitly supplied provider. Remove unintended
earlier sources and verify the chain used by the deployed SDK version.

This ambient path is also suitable for a controller's installation identity. It
must not infer the IAM role for a delegated target namespace operation.

## EKS Pod Identity

EKS Pod Identity associates a Kubernetes ServiceAccount with an IAM role through
the EKS control plane. A node agent exposes credentials through a container
credential endpoint that supported AWS SDKs discover.

It does not expose the same portable OIDC exchange contract as IRSA. A
multi-tenant controller cannot select another namespace's EKS Pod Identity by
pointing its own default credential chain at the controller pod's endpoint. Use
TokenRequest and an explicit web-identity trust path for delegated target
identity unless a purpose-built broker is defined.

## Explicit federation

An ordinary workload can explicitly project a token and configure
`AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. The AWS default provider chain
can then obtain the workload's own role session.

A multi-tenant controller must not change those environment variables per
reconciliation. Prefer an explicit SDK web-identity provider configured with the
selected target role ARN, a renewable target token supplier, and a traceable
role session name. If calling `AssumeRoleWithWebIdentity` directly, use the
SDK's anonymous or no-signing STS configuration where required; the operation
does not require controller AWS credentials.

## Controller identity

A controller that needs AWS access for controller-owned operations can use:

- IRSA.
- EKS Pod Identity.
- Explicit web identity configuration.
- EC2 instance or ECS task credentials where appropriate.
- Static access keys, which are discouraged.

The AWS default credential provider chain is appropriate for these
controller-owned calls when its precedence is understood. It must not select a
target namespace role.

## Delegated target identity

For `serviceAccountRef`, the controller should:

1. Resolve the ServiceAccount in the target namespace.
2. Configure a renewable token supplier for `sts.amazonaws.com` or the audience required by the role trust policy.
3. Resolve the target role ARN.
4. Construct an explicit web-identity credential provider, or call `AssumeRoleWithWebIdentity` with unsigned STS requests for a bounded operation.
5. Construct the target AWS service client with the refreshable provider or returned role session.

Do not use the controller's default credential chain for step 4 or step 5.
Credentials returned by STS expire. A client that can outlive one role session
must use a provider that requests a fresh Kubernetes assertion and renews the
STS session. Fixed returned credentials are suitable only for an operation that
is guaranteed to finish before they expire.

The exceptions are an explicitly selected `controllerIdentity` mode or a
controller policy that explicitly allows fallback for the failed credential
stage. In either case, deliberately reuse the already established controller
credential after policy evaluation; do not invoke the default provider chain as
target discovery or implicit fallback.

### Target role resolution

Use this order:

1. Explicit role ARN in namespace configuration.
2. `eks.amazonaws.com/role-arn` on the target ServiceAccount.
3. Fail if no role ARN is resolved.

Namespace configuration explicitly overrides the annotation. The annotation is
a useful convention even outside EKS when the controller performs the STS
exchange itself.

The role trust policy remains the final identity boundary. Restrict `sub` to the
intended namespace and ServiceAccount, restrict `aud`, and avoid wildcard trust
across tenant namespaces.

### Role chaining

Some designs first use `AssumeRoleWithWebIdentity` and then call `AssumeRole` for
a second role. This is role chaining, not part of the initial web identity
exchange.

Use role chaining only when it provides a required account or authorization
boundary. Account for its session-duration and policy constraints, and include
both roles in audit analysis.

## Cross-cloud clusters

An IAM OIDC provider can represent a suitable issuer from GKE, AKS, another
Kubernetes distribution, or an on-premises cluster. The cluster does not need to
run on AWS.

The issuer metadata and signing keys must remain reachable and stable. Explicit
`AssumeRoleWithWebIdentity` is normally used because EKS admission and Pod
Identity mechanisms are not available on other clusters.

## Private issuers

AWS validates web identity tokens through the configured IAM OIDC provider and
its issuer metadata and keys. The HTTPS issuer, discovery document, and JWKS
endpoint must be publicly reachable; an issuer available only on a private
network cannot be used for direct IAM OIDC federation. AWS has no uploaded-JWKS
alternative comparable to Google Workload Identity Federation. Publishing only
the required issuer endpoints through a trusted public boundary can avoid
exposing the Kubernetes API itself.

Plan for issuer TLS, signing-key rotation, and the lifetime of IAM OIDC provider
configuration. AWS currently limits a provider JWKS to 100 RSA keys and 100 EC
keys; exceeding the applicable limit causes token validation failures. Keep
active and retiring rotation keys within that limit. A trust policy should not
broaden subjects merely to work around issuer-management problems.

## Static target credentials

A compatibility `secretRef` can contain `aws_access_key_id`,
`aws_secret_access_key`, and, for a temporary credential, `aws_session_token`.
Construct a target-scoped static credential provider directly from the Secret.
Do not set process-wide `AWS_*` variables.

Long-lived IAM user keys are not recommended. Migrate by establishing IAM OIDC
provider trust, adding a narrowly scoped role trust policy, and replacing the
Secret reference with `serviceAccountRef`.

## Amazon Cognito user authentication

Amazon Cognito's token endpoint currently documents `client_secret_basic` and
`client_secret_post` for confidential app clients. It does not document a
Kubernetes workload token, IAM role session, or `private_key_jwt` as client
authentication for authorization-code redemption.

Where appropriate, use a Cognito public app client with Authorization Code with
PKCE so that no client secret is placed in the workload. This is an OAuth client
design choice, not AWS API workload identity. A backend can separately use
IRSA, EKS Pod Identity, or explicit web identity for its own AWS API calls.

## Troubleshooting

Check these boundaries in order:

1. The projected token has the expected issuer, subject, audience, and expiry.
2. IAM OIDC provider configuration matches the issuer.
3. The role trust policy permits the exact web identity claims.
4. STS returned the intended role ARN and session.
5. The role policy and resource policy authorize the AWS operation.
6. The target service client received the explicit STS provider/session, or an explicitly policy-approved controller credential, rather than an implicit default-chain fallback.

Use a target-specific role session name for auditability without placing
sensitive tenant data in it. Do not log the web identity token, STS secret access
key, session token, or static Secret values.

## Primary references

- [IAM OIDC identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [STS AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html)
- [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [Amazon Cognito token endpoint](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html)
