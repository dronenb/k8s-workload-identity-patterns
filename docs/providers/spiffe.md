# SPIFFE workload identity

SPIFFE provides provider-neutral workload identities through SPIFFE IDs and
SPIFFE Verifiable Identity Documents (SVIDs). A SPIFFE-native workload obtains
its own SVIDs directly from the Stable SPIFFE Workload API, without Kubernetes
projected ServiceAccount tokens. Trusted infrastructure acting for another
workload can obtain that workload's SVIDs through the Incubating SPIFFE Broker
API.

This guide applies the models in [Kubernetes workload identity](../workload-identity.md)
and [multi-tenant identity](../multi-tenant-identity.md).

## Stability

| Specification | Stability | Use |
| --- | --- | --- |
| SPIFFE ID and SVID | Stable | Identify a workload within a trust domain. |
| Workload Endpoint and Workload API | Stable | Deliver identity to the local workload. |
| Broker Endpoint and Broker API | Incubating | Deliver identity to trusted infrastructure acting for a referenced workload. |
| X.509-SVID and JWT-SVID | Stable | Certificate and JWT workload credentials. |
| WIT-SVID profile | Incubating | Optional proof-of-possession workload credential. |

Incubating specifications avoid breaking changes but can still change based on
implementation experience. Gate Broker API support behind an explicit opt-in,
record the implemented specification revision, and do not imply that every
SPIFFE or SPIRE deployment supports it.

## Identity choices

| Choice | Result |
| --- | --- |
| Local Workload API | The caller receives its own SVIDs and trust bundles. This is the primary identity source for a SPIFFE-native workload. |
| Broker API | An authorized broker receives SVIDs and bundles for one referenced workload. Use this for delegated tenant identity. |
| Exported key or token | Identity material is copied into a file or Secret. Avoid when the API can provide and rotate it. |

An SVID can authenticate directly to a SPIFFE-aware peer. It can also be used as
an assertion in another provider's exchange when that provider explicitly
supports the SVID format, issuer, and claims. SPIFFE does not automatically turn
an SVID into a Google, Azure, or AWS credential.

## SPIFFE IDs and trust domains

A SPIFFE ID is a URI with a trust domain and workload-specific path:

```text
spiffe://cluster.example.com/ns/payments/sa/reconciler
```

The path convention is operator-defined. A path that resembles a Kubernetes
namespace and ServiceAccount is not proof of those Kubernetes properties unless
the trust-domain operator's attestation and registration policy establishes
that meaning.

Trust bundles contain the public keys used to validate SVIDs. Federation between
SPIFFE trust domains distributes selected foreign bundles; it does not merge the
trust domains or make similarly named identities equivalent.

## Explicit local configuration

The Workload Endpoint can be configured directly in code or located through the
well-known `SPIFFE_ENDPOINT_SOCKET` environment variable. Its value is an RFC
3986 URI using `unix` or `tcp`:

```yaml
env:
  - name: SPIFFE_ENDPOINT_SOCKET
    value: unix:///run/spire/sockets/agent.sock
```

Mount only the intended endpoint socket into the pod. Unix domain sockets are
preferred. A TCP Workload Endpoint must remain local to one host and use a
network mechanism that lets the implementation strongly identify the caller.

The Workload Endpoint does not require a Kubernetes ServiceAccount token or any
other client credential. The implementation identifies the caller out of band,
such as through kernel or orchestrator attestation. Clients must use a
conforming SPIFFE library, which supplies the required `workload.spiffe.io:
true` gRPC metadata.

```text
Local workload
  -> SPIFFE_ENDPOINT_SOCKET
  -> Workload API identifies the local caller
  -> workload's SVIDs and trust bundles
```

Applications should keep streaming Workload API connections open for X.509-SVID,
WIT-SVID, and bundle updates so rotations and redactions are applied promptly.
JWT-SVID retrieval is a unary request; request a new audience-bound JWT-SVID
before the current one expires.

## SVID profiles

### X.509-SVID

An X.509-SVID contains a SPIFFE ID in the certificate and has a corresponding
private key. Use it for mutual TLS or another protocol that validates the peer's
SPIFFE ID and trust-domain bundle.

The Workload API streams replacements. Keep private keys in memory and avoid
writing them to a Kubernetes Secret or shared filesystem.

### JWT-SVID

A JWT-SVID is a short-lived bearer token requested for a specific audience. The
relying party must validate its signature, expiry, audience, and SPIFFE ID using
the correct trust-domain JWT bundle.

Request only the audience needed by the relying party. A JWT-SVID issued for one
audience must not be reused with another service.

### WIT-SVID

WIT-SVID is an optional, Incubating proof-of-possession profile. Implementations
can return `Unimplemented`, and applications must not require it without an
explicit compatibility check and opt-in.

## Controller identity

A controller that needs its own SPIFFE identity should use its local Workload
Endpoint through explicit configuration or `SPIFFE_ENDPOINT_SOCKET`:

```text
Controller process
  -> local Workload API
  -> controller SVID
  -> controller-owned operation
```

This identity can authenticate the controller to shared SPIFFE-aware services.
It can also bootstrap the X.509-SVID used to authenticate to a Broker Endpoint.
It must not be reused as a target namespace SVID merely because it is available
through the controller's local socket.

## Delegated target identity

The Workload API identifies the process connected to it. A controller calling
that API receives the controller identity, not the identity of a target
namespace. Tenant identity therefore uses the Broker API rather than the local
Workload API.

The Broker Endpoint can be configured directly or located with
`SPIFFE_BROKER_SOCKET`:

```yaml
env:
  - name: SPIFFE_ENDPOINT_SOCKET
    value: unix:///run/spire/sockets/agent.sock
  - name: SPIFFE_BROKER_SOCKET
    value: unix:///run/spire/sockets/broker.sock
```

The first endpoint supplies the controller or broker's own X.509-SVID. The
second endpoint accepts authorized, on-behalf-of requests. Do not treat the two
variables as interchangeable fallbacks.

```text
Controller obtains its own X.509-SVID from Workload API
  -> mutually authenticates to SPIFFE_BROKER_SOCKET
  -> Broker API authorizes controller and workload reference
  -> provider resolves reference and verifies existence and SVID entitlement
  -> target SVIDs and target-specific bundles
```

Both the Broker Endpoint and Broker API are Incubating.

### Broker endpoint security

The Broker Endpoint requires mutual TLS with X.509-SVIDs. The client must verify
the expected SPIFFE ID of the provider, and the provider must apply an allow-only
authorization policy to the broker's SPIFFE ID. Network or socket accessibility
alone is not sufficient authorization.

A Broker Endpoint can use a Unix domain socket or a TCP endpoint. Prefer a local
Unix socket where possible. A remote TCP endpoint remains mutually authenticated
and should be exposed only to designated brokers.

Conforming clients send the required `broker.spiffe.io: true` gRPC metadata.
Use a SPIFFE Broker API library rather than implementing an unauthenticated
generic gRPC client.

### Kubernetes workload references

The Incubating Broker API defines a `KubernetesObjectReference` containing:

- The resource plural and API group, using `core` for core resources.
- An optional key with name and, for namespaced objects, namespace.
- An optional Kubernetes object UID.

The resource type is required, and at least one of key or UID is required. Valid
forms are key-only, UID-only, and key plus UID. A key for a cluster-scoped
resource has an empty namespace.

An illustrative target reference is:

```yaml
type:
  plural: serviceaccounts
  group: core
key:
  namespace: payments
  name: reconciler
uid: 2b281a86-7d16-4e55-b336-1c20d72c76a8
```

This fragment illustrates the Broker API message and is not a Kubernetes CRD.
Including both the key and UID prevents a deleted and recreated object with the
same name from silently retaining the old reference. The server independently
resolves the reference and verifies that the object exists and is entitled to
SVIDs; it must not trust client-provided reference attributes by themselves.

The mapping from a Kubernetes object to a SPIFFE ID is implementation-defined.
Document that mapping and do not assume a universal
`spiffe://.../ns/.../sa/...` convention.

Controller policy should normally require a namespaced target reference to use
the target resource's namespace. The existing `key.namespace` field can name
another namespace in the same Kubernetes control plane, so any cross-namespace
use requires explicit controller policy and Broker Endpoint authorization. A
Kubernetes object reference cannot cross Kubernetes control planes.

### Per-target isolation and rotation

Broker API calls are scoped to one concrete workload reference. Keep each
target's SVIDs, private keys, and bundles isolated from every other target and
from the controller identity.

X.509-SVID and bundle subscriptions stream complete replacement state. If an
identity or bundle disappears from an update, stop using it. When the referenced
object no longer exists or no longer matches a pinned UID, close its operations
and discard its material promptly.

For JWT-SVIDs, request the target's required audience explicitly. Do not cache a
JWT-SVID across workloads merely because they map to the same SPIFFE ID.

## Direct use and cloud exchange

For a SPIFFE-aware target service, construct the TLS or JWT client explicitly
with the selected target SVID and target bundles. Do not feed target SVIDs into a
process-wide controller credential chain.

For a cloud exchange, construct a target-scoped cloud credential from the target
SVID only when the cloud provider supports that format and trust relationship.
The cloud provider's authorization remains a separate boundary. See the
[Google Cloud](google-cloud.md), [Microsoft Azure](azure.md), and
[AWS](aws.md) guides for provider-specific behavior.

An SVID exchange failure follows the same controller fallback policy as other
target credential sources. It fails closed unless controller policy explicitly
allows fallback for that failure stage. A target authorization denial must not
silently retry with the controller SVID or controller cloud identity.

## Avoid exported static identity

Exporting an X.509-SVID private key or JWT-SVID to a Kubernetes Secret can extend
credential access beyond the referenced workload's lifecycle. Exporting a trust
bundle does not itself extend workload authorization, but a static copy can
become stale as trust roots change. Prefer a mounted endpoint and an API client
that streams X.509 and bundle updates and explicitly renews JWT-SVIDs.

If a legacy application requires files, use a purpose-built delivery component
that applies updates atomically, constrains filesystem access, stops delivering
after workload deletion, and removes stale material. Do not present this as
equivalent to the Workload or Broker API security model.

## Troubleshooting

Check these boundaries in order:

1. The expected endpoint URI and socket are available to the client.
2. A Workload API call is attributed to the intended local process.
3. Broker mTLS validates both the broker and provider SPIFFE IDs.
4. Broker authorization permits the caller, reference type, namespace, audience, and requested identity.
5. The provider resolves the Kubernetes object and validates its UID and entitlement.
6. The client applies streamed rotations and redactions only to the referenced target.
7. The relying party trusts the correct SPIFFE bundle or explicitly supports the cloud exchange.

Do not log SVID private keys, JWT-SVIDs, WIT-SVID private keys, or complete
Broker API responses containing credential material.

## Primary references

- [SPIFFE ID and SVID](https://spiffe.io/docs/latest/spiffe-specs/spiffe-id/)
- [SPIFFE Workload Endpoint](https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_endpoint/)
- [SPIFFE Workload API](https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_api/)
- [SPIFFE Broker Endpoint](https://spiffe.io/docs/latest/spiffe-specs/spiffe_broker_endpoint/)
- [SPIFFE Broker API](https://spiffe.io/docs/latest/spiffe-specs/spiffe_broker_api/)
- [SPIFFE specification stability](https://spiffe.io/docs/latest/spiffe-specs/stability/)
- [SPIFFE Federation](https://spiffe.io/docs/latest/spiffe-specs/spiffe_federation/)
