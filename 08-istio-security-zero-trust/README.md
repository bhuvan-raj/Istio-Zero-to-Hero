# Module 08: Istio Security and Zero Trust

## 📌 Overview

In a microservices architecture, services communicate with one another over the network. Without proper security controls, unauthorized workloads may access sensitive services, and attackers may intercept or impersonate service-to-service communication.

**Istio security** provides mechanisms to authenticate workloads, encrypt service-to-service communication, and control which workloads are allowed to access specific services.

Istio supports a **Zero Trust** security model: workloads should not be trusted simply because they run inside the same cluster or network. Access should be verified and explicitly authorized.

In this module, you will learn how Istio uses workload identity, mutual TLS (mTLS), authentication policies, and authorization policies to secure microservices.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What Zero Trust security means
- How Istio identifies workloads
- What workload identity and SPIFFE identities are
- The difference between authentication and authorization
- How mutual TLS encrypts service-to-service communication
- The difference between `PeerAuthentication` and `RequestAuthentication`
- How to configure `AuthorizationPolicy`
- How to restrict access between microservices
- How to troubleshoot mTLS and authorization failures

## 📚 Prerequisites

- A working Kubernetes cluster
- Istio installed with a supported data plane
- `kubectl` and `istioctl` installed
- Basic knowledge of Kubernetes Deployments, Services, and namespaces
- Completion of Modules 04–07

---

## 1. What Is Zero Trust?

Zero Trust is a security approach based on the principle that no workload or user should be trusted automatically based on network location alone.

Traditional network security often relies heavily on a trusted internal network. However, a compromised workload inside a cluster could potentially attempt to communicate with other services.

Zero Trust aims to reduce this risk by verifying identities, encrypting communication, and enforcing access policies.

### Core principles

- **Verify explicitly:** Authenticate workloads and users where required.
- **Use least privilege:** Grant only the permissions necessary to perform a task.
- **Assume breach:** Design controls that limit the impact of a compromised workload.
- **Protect communication:** Encrypt traffic and validate identities.
- **Continuously monitor:** Observe access attempts and investigate suspicious behavior.

### Example

Consider an application with three microservices:

```text
Frontend
   |
   v
Order Service
   |
   v
Payment Service
```

Without appropriate access controls, a compromised workload might attempt to call the Payment Service directly.

With Istio security policies, you can require authenticated workload communication and explicitly allow only the Order Service to access the Payment Service.

---

## 2. How Istio Provides Security

Istio provides security capabilities through workload identities, certificates, Envoy proxies, and security policies.

```text
              Istio Control Plane
                     |
                     | Configuration
                     | Certificates
                     v
+------------------------------------------+
|              Kubernetes Cluster         |
|                                          |
|  Frontend Pod             Backend Pod    |
|  +-------------+          +-------------+|
|  | Application |          | Application ||
|  +-------------+          +-------------+|
|  | Envoy Proxy | =======> | Envoy Proxy ||
|  +-------------+  mTLS    +-------------+|
|                                          |
+------------------------------------------+
```

Istio's control plane manages workload identity and distributes configuration and certificates to proxies. Envoy proxies enforce supported authentication, encryption, and authorization policies.

The exact behavior depends on the installation mode, data plane, and configured policies.

### Important security resources

| Resource | Purpose |
|---|---|
| `PeerAuthentication` | Configures mTLS requirements for incoming workload connections |
| `RequestAuthentication` | Configures JWT authentication for incoming requests |
| `AuthorizationPolicy` | Allows or denies access based on configured identities and request attributes |
| `DestinationRule` | Can configure TLS settings for outbound traffic to a destination |
| `Gateway` | Configures edge traffic handling, including TLS settings where applicable |

**Important:** These resources serve different purposes. Enabling mTLS does not automatically mean that every authenticated workload is authorized to access every service.

---

## 3. Workload Identity

### What is workload identity?

A workload identity is an identity assigned to a running workload so that other components can verify which workload is communicating.

In Kubernetes, workloads commonly use ServiceAccounts. Istio can use Kubernetes ServiceAccount identities to establish workload identity.

For example, a workload might have an identity resembling:

```text
cluster.local/ns/istio-security/sa/order-service
```

This SPIFFE-style identity identifies:

- Trust domain: `cluster.local`
- Namespace: `istio-security`
- ServiceAccount: `order-service`

The exact identity depends on your Istio trust domain and workload configuration.

### Why is workload identity important?

Without workload identity, access controls may rely heavily on IP addresses or network location. These are often insufficient to reliably identify a workload.

With workload identity, Istio policies can grant access based on the authenticated service identity.

For example, you can allow a workload running as the `order-service` ServiceAccount to call the Payment Service while denying other workloads.

### Create a dedicated ServiceAccount

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: istio-security
```

Apply it:

```bash
kubectl apply -f service-account.yaml
```

To use the identity, configure the workload's Pod template:

```yaml
spec:
  template:
    spec:
      serviceAccountName: order-service
```

The workload must actually run using this ServiceAccount for Istio to identify it accordingly.

---

## 4. Authentication vs Authorization

Authentication and authorization are related, but they answer different questions.

| Concept | Question | Istio example |
|---|---|---|
| Authentication | Who are you? | mTLS workload identity or a validated JWT |
| Authorization | What are you allowed to do? | An AuthorizationPolicy permitting a specific workload |

### Authentication

Authentication verifies the identity of a communicating workload or request.

Examples:

- mTLS authenticates workloads using certificates.
- JWT authentication validates a token's identity and claims according to configured requirements.

### Authorization

Authorization determines whether an authenticated identity or request is allowed to access a resource.

For example:

- Allow the Order Service to access the Payment Service.
- Deny other workloads access to the Payment Service.
- Allow only authenticated users with an appropriate JWT claim to call a protected API.

**Remember:** Authentication establishes identity. Authorization decides what that identity can do.

---

## 5. Mutual TLS (mTLS)

### What is TLS?

Transport Layer Security (TLS) provides encryption and integrity protection for communication. Standard server-authenticated TLS allows a client to verify the server's identity.

### What is mutual TLS?

Mutual TLS authenticates both sides of a connection using certificates.

In Istio, mTLS can allow workloads to verify one another's identities while encrypting traffic between their Envoy proxies.

### How mTLS works

1. A workload sends a request to another workload.
2. The source Envoy proxy establishes a TLS connection to the destination proxy.
3. The proxies present and validate their certificates.
4. The connection uses the negotiated TLS session to protect the communication.
5. Istio policies can use the authenticated workload identity for authorization.

### Why use mTLS?

- Encrypts traffic between participating workloads.
- Helps protect against eavesdropping and tampering.
- Provides cryptographic workload authentication.
- Enables identity-based authorization policies.

mTLS does not, by itself, determine whether a workload should be permitted to perform a particular application operation.

---

## 6. Lab: Verify Istio mTLS

Istio commonly uses automatic mutual TLS when workloads have compatible sidecars and the required configuration is in place. However, the effective behavior depends on your installation and policies.

Check the status of the proxies:

```bash
istioctl proxy-status
```

Inspect authentication policies:

```bash
kubectl get peerauthentication -A
```

Inspect destination rules:

```bash
kubectl get destinationrule -A
```

Use Istio's authentication policy checks to inspect the security configuration for a workload:

```bash
istioctl x describe pod <pod-name> -n <namespace>
```

Replace the placeholders with an actual Pod name and namespace.

### Enforce strict mTLS in a namespace

Create `peer-authentication.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-security
spec:
  mtls:
    mode: STRICT
```

Apply it:

```bash
kubectl apply -f peer-authentication.yaml
```

This policy requires participating workload traffic in the namespace to use Istio mTLS. It is appropriate only when the relevant workloads and communication paths support the required mesh configuration.

### Verify the policy

```bash
kubectl get peerauthentication -n istio-security
kubectl describe peerauthentication default -n istio-security
```

**Important:** Do not apply `STRICT` mTLS blindly to an existing production namespace. First identify workloads without compatible proxies, external clients, gateways, and other communication paths that could be affected.

---

## 7. PeerAuthentication

`PeerAuthentication` controls how a workload accepts incoming connections with respect to mutual TLS.

The principal modes are:

| Mode | Meaning |
|---|---|
| `UNSET` | Inherits the applicable parent or mesh-level setting |
| `DISABLE` | Disables Istio mTLS for the applicable inbound traffic |
| `PERMISSIVE` | Accepts both plaintext and Istio mTLS traffic |
| `STRICT` | Requires Istio mTLS for the applicable inbound traffic |

### Example: Permissive mode

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: backend-auth
  namespace: istio-security
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: PERMISSIVE
```

This policy selects workloads labelled `app: backend` and allows both plaintext and Istio mTLS connections.

Permissive mode can help with migration, but it does not provide the same enforcement as strict mTLS.

### Example: Strict mode for a workload

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: backend-auth
  namespace: istio-security
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: STRICT
```

This requires Istio mTLS for incoming connections to the selected workloads, subject to the applicable configuration and supported data plane.

**Note:** A `PeerAuthentication` policy controls incoming connection authentication. It does not create an allowlist of application callers; use `AuthorizationPolicy` for access control.

---

## 8. RequestAuthentication and JWT

### What is JWT authentication?

A JSON Web Token (JWT) is a token format commonly used to carry signed claims about a user or client.

A JWT can contain claims such as:

- Issuer (`iss`)
- Subject (`sub`)
- Audience (`aud`)
- Expiration (`exp`)

A JWT must be validated according to the configured issuer, signing keys, and relevant validation requirements.

### What does RequestAuthentication do?

Istio's `RequestAuthentication` configures JWT validation for incoming requests to selected workloads.

Create `request-authentication.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: backend-jwt
  namespace: istio-security
spec:
  selector:
    matchLabels:
      app: backend
  jwtRules:
    - issuer: "https://issuer.example.com"
      jwksUri: "https://issuer.example.com/.well-known/jwks.json"
```

Replace the example issuer and JWKS URL with values from your actual identity provider.

Apply it:

```bash
kubectl apply -f request-authentication.yaml
```

### Important behavior

`RequestAuthentication` validates JWTs when they are supplied, but a request without a JWT can still be accepted by this policy alone.

If your service must reject requests without an authenticated JWT, combine `RequestAuthentication` with an appropriate `AuthorizationPolicy`.

Do not use the example issuer or URL in production.

---

## 9. AuthorizationPolicy

### What is AuthorizationPolicy?

An `AuthorizationPolicy` controls access to workloads based on configured identities and request attributes.

It can apply to a namespace, selected workloads, or the mesh, depending on its scope.

An authorization policy can use attributes such as:

- Source namespace
- Source principal
- Request paths
- HTTP methods
- JWT claims, when properly authenticated

Istio supports policy actions including `ALLOW`, `DENY`, `AUDIT` where supported, and `CUSTOM` for supported external authorization integrations.

### Lab: Allow only the Order Service to call the Backend

Suppose the backend should accept requests only from a workload running with the `order-service` ServiceAccount in the `istio-security` namespace.

Create `authorization-policy.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-allow-order-service
  namespace: istio-security
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/istio-security/sa/order-service"
```

Apply it:

```bash
kubectl apply -f authorization-policy.yaml
```

### Explanation

- `selector`: Selects the backend workloads labelled `app: backend`.
- `action: ALLOW`: Permits requests matching the specified rule.
- `principals`: Matches the authenticated source workload identity.
- `source`: Defines conditions that the caller must satisfy.

The principal must match the actual identity presented by the caller. If your trust domain differs from `cluster.local`, adjust the principal accordingly.

### Important behavior

Once an `ALLOW` policy applies to a workload, requests that do not match an applicable allow rule are denied, subject to Istio's policy evaluation rules.

This means other callers, including the earlier frontend test client, may no longer be able to reach the backend unless they match an allow rule.

**Best practice:** Validate the source identity and test access with both permitted and denied workloads before applying policies to production services.

---

## 10. Restrict Access by HTTP Path and Method

Authorization policies can also restrict which HTTP operations a workload may perform.

For example, allow the Order Service to make `GET` requests to `/api/orders` on the backend.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-api-access
  namespace: istio-security
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/istio-security/sa/order-service"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/orders"]
```

This is an illustrative policy for an HTTP application. Replace the example path with a real application endpoint.

For path and method authorization, ensure that Istio recognizes the traffic as HTTP. If traffic is treated as opaque TCP, HTTP-specific attributes cannot be evaluated as expected.

### Example use cases

- Allow `GET` requests but deny `DELETE` requests.
- Allow a workload to access only specific API paths.
- Restrict access to administrative endpoints.
- Permit one microservice to call another while blocking unrelated workloads.

---

## 11. Combine mTLS and Authorization

mTLS and authorization are complementary controls.

```text
Request
   |
   v
mTLS authentication
   |
   v
Workload identity verified
   |
   v
AuthorizationPolicy evaluation
   |
   +---- Allowed ----> Application
   |
   +---- Denied -----> Access rejected
```

For example:

1. mTLS establishes an authenticated workload identity.
2. `PeerAuthentication` controls whether mTLS is required.
3. `AuthorizationPolicy` determines whether that identity may access the selected service.
4. `RequestAuthentication` can validate JWTs for user or client requests.

These mechanisms answer different security questions and should be configured according to your application's trust boundaries.

---

## 12. Common Security Troubleshooting

| Problem | Possible cause | What to check |
|---|---|---|
| Requests fail after enabling STRICT mTLS | A caller or communication path does not use compatible Istio mTLS | Check sidecar injection, proxy status, and applicable PeerAuthentication policies |
| Requests return HTTP 403 | An AuthorizationPolicy denies the request | Inspect policy selectors, rules, and source identity |
| Requests return HTTP 401 | A JWT is missing or fails validation, or the application itself rejects it | Check RequestAuthentication, token issuer, signature, and expiration |
| JWT is accepted when it should be required | RequestAuthentication alone does not require a JWT on every request | Add an appropriate AuthorizationPolicy |
| Allowed caller is denied | The principal does not match the actual workload identity | Check trust domain, namespace, and ServiceAccount |
| HTTP path rules do not work | Traffic is not recognized as HTTP or the path does not match | Check service port naming, protocol detection, and policy rules |
| Policy has no effect | The selector matches no workloads or the policy is in the wrong scope | Check labels and namespace |
| Service communication breaks after a policy change | An ALLOW policy excludes legitimate callers | Review all permitted sources and test both positive and negative cases |

### Useful commands

List security policies:

```bash
kubectl get peerauthentication -A
kubectl get requestauthentication -A
kubectl get authorizationpolicy -A
```

Inspect a workload:

```bash
kubectl get pods -n istio-security --show-labels
istioctl proxy-status
istioctl x describe pod <pod-name> -n istio-security
```

Analyze the configuration:

```bash
istioctl analyze -n istio-security
```

Inspect proxy logs:

```bash
kubectl logs <pod-name> -n istio-security -c istio-proxy
```

Replace `<pod-name>` with the name of the affected Pod.

---

## 13. Security Best Practices

- Use dedicated Kubernetes ServiceAccounts for application workloads.
- Enable mTLS where supported and validate the impact before enforcing `STRICT`.
- Use AuthorizationPolicy to implement least-privilege access.
- Do not assume that being inside the cluster makes a workload trustworthy.
- Validate JWT issuer, signing keys, expiration, and audience requirements as appropriate.
- Require authentication explicitly when a service must not accept anonymous requests.
- Avoid broad allow rules that unintentionally grant access to every workload.
- Review namespace and workload selectors carefully.
- Keep sensitive credentials out of container images and source control.
- Monitor denied requests and investigate unexpected access patterns.
- Test both allowed and denied requests before deploying policy changes.
- Keep Istio and its supported components updated.
- Remember that mesh policies do not replace Kubernetes RBAC, network controls, application authorization, or secure secret management.

---

## 14. Hands-On Exercises

- [ ] Create a dedicated ServiceAccount for a workload.
- [ ] Inspect the workload identity used by Istio.
- [ ] Verify the current mTLS configuration.
- [ ] Apply a `PeerAuthentication` policy in `STRICT` mode in a test namespace.
- [ ] Compare `PERMISSIVE` and `STRICT` behavior.
- [ ] Configure a `RequestAuthentication` policy using a real test identity provider.
- [ ] Create an `AuthorizationPolicy` that allows one workload identity.
- [ ] Test a permitted request and a denied request.
- [ ] Restrict an HTTP endpoint by path and method.
- [ ] Diagnose a 403 error caused by an authorization policy.
- [ ] Explain why mTLS alone does not provide least-privilege access.

Use a dedicated test namespace for these exercises so that you can safely change security policies without disrupting other workloads.

---

## 15. Interview Questions

1. What is Zero Trust security?
2. How does Istio implement workload identity?
3. What is the difference between TLS and mTLS?
4. What is the difference between PeerAuthentication and RequestAuthentication?
5. What is the difference between authentication and authorization?
6. What is the purpose of AuthorizationPolicy?
7. Does RequestAuthentication reject every request without a JWT?
8. How do you allow only one microservice to access another?
9. What is the difference between PERMISSIVE and STRICT mTLS?
10. Why might a request return HTTP 403 after an AuthorizationPolicy is applied?
11. How does Istio use workload identities in authorization rules?
12. Why is mTLS not a replacement for application-level authorization?
13. How would you troubleshoot a service that stops communicating after STRICT mTLS is enabled?
14. How does Istio security complement Kubernetes RBAC?

---

## 16. Key Takeaways

- Zero Trust means access should be verified rather than granted solely because a workload is inside a trusted network.
- Workload identity allows Istio to identify services using authenticated identities.
- mTLS encrypts traffic and authenticates participating workloads.
- PeerAuthentication controls inbound mTLS requirements.
- RequestAuthentication configures JWT validation.
- AuthorizationPolicy controls which identities and requests are allowed.
- Authentication and authorization are separate but complementary.
- Istio security should be combined with Kubernetes RBAC, secure secret management, and application-level security.

## 📖 Official Documentation

- [Istio Security Concepts](https://istio.io/latest/docs/concepts/security/)
- [Mutual TLS Migration](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
- [PeerAuthentication Reference](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- [RequestAuthentication Reference](https://istio.io/latest/docs/reference/config/security/request_authentication/)
- [Authorization Policy Reference](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- [Authentication Policy](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/)
- [Authorization Policy](https://istio.io/latest/docs/tasks/security/authorization/authz-policy/)

## ➡️ Next Module

Continue to [Module 09: Observability](../09-observability/README.md) to learn how to monitor service traffic, analyze metrics, inspect distributed traces, and troubleshoot microservices using Istio telemetry.
