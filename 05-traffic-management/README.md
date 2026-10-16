# Module 05: Istio Traffic Management

## 📌 Overview

Istio Traffic Management allows you to control how requests flow between microservices without modifying application code.

In Kubernetes, a Service distributes traffic across its available Pods. Istio extends this capability by allowing you to control traffic based on HTTP paths, headers, service versions, weights, timeouts, and retry policies.

Istio uses **Envoy proxies** to apply traffic management rules inside the service mesh.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What traffic management means in Istio
- The difference between VirtualService and DestinationRule
- How to route requests based on paths and headers
- How to distribute traffic between application versions
- How to configure timeouts and retries
- How to configure load balancing and circuit breaking
- How to validate and troubleshoot traffic rules

## 📚 Prerequisites

- A working Kubernetes cluster
- Istio installed in the cluster
- The `istioctl` CLI installed
- Basic knowledge of Deployments, Services, Pods, and YAML
- Familiarity with Module 04: Your First Istio Application

---

## 1. What Is Traffic Management?

Traffic management is the process of controlling how requests travel between services.

Consider an application with two versions of a backend service:

- `backend-v1`: Existing stable version
- `backend-v2`: New version containing recent changes

Without Istio traffic management, you generally rely on Kubernetes Service load balancing and application-level routing.

With Istio, you can define rules such as:

- Send all requests to version 1.
- Send 10% of requests to version 2.
- Route requests with a particular HTTP header to version 2.
- Route `/api/users` to one service and `/api/orders` to another.
- Retry failed requests or stop waiting after a specified timeout.

### How it works

1. A client sends a request to a service.
2. The Envoy proxy intercepts the request.
3. Envoy evaluates the traffic configuration received from Istio.
4. The request is forwarded to the selected destination.
5. The response returns to the client.

Istio's control plane configures the proxies, while Envoy proxies handle the actual traffic.

---

## 2. Core Traffic Management Resources

| Resource | Purpose |
|---|---|
| VirtualService | Defines how requests are routed |
| DestinationRule | Defines destination policies and service subsets |
| Gateway | Configures traffic entering or leaving the mesh |
| ServiceEntry | Registers external services in the mesh |
| Sidecar | Restricts the set of services a workload's proxy can reach |

### VirtualService

A `VirtualService` defines routing rules for a hostname. It can route requests based on URI paths, HTTP headers, query parameters, and other supported conditions.

It can also configure traffic weights, redirects, rewrites, timeouts, and retries.

**Remember:** VirtualService answers, "Where should this request go?"

### DestinationRule

A `DestinationRule` defines policies applied after routing has selected a destination. It supports service subsets, load-balancing policies, connection pools, outlier detection, and TLS settings.

**Remember:** DestinationRule answers, "What policies should apply to the selected destination?"

### Gateway

A `Gateway` configures an edge proxy to accept or send traffic using specified ports, protocols, hosts, and TLS settings.

A Gateway normally works with a VirtualService to route incoming requests to application services.

### ServiceEntry

A `ServiceEntry` adds an external service or other non-Kubernetes destination to Istio's service registry so that mesh traffic to it can be managed using Istio policies.

---

## 3. Lab Environment

For this module, we will create two backend versions and one frontend client.

```text
frontend
   |
   v
Envoy Proxy
   |
   v
VirtualService
   |
   +---- 90% ----> backend-v1
   |
   +---- 10% ----> backend-v2
```

Both backend versions are exposed through the same Kubernetes Service, `backend`.

Create a namespace with automatic sidecar injection enabled:

```bash
kubectl create namespace istio-traffic
kubectl label namespace istio-traffic istio-injection=enabled
```

If the namespace already exists, use the label command directly.

### Create the application

Create `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v1
  namespace: istio-traffic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Response from backend-v1"
          ports:
            - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v2
  namespace: istio-traffic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v2
  template:
    metadata:
      labels:
        app: backend
        version: v2
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Response from backend-v2"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-traffic
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 5678
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: istio-traffic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: curl
          image: curlimages/curl:8.10.1
          command: ["sh", "-c", "sleep 3650d"]
```

Apply the manifest:

```bash
kubectl apply -f application.yaml
kubectl rollout status deployment/backend-v1 -n istio-traffic
kubectl rollout status deployment/backend-v2 -n istio-traffic
kubectl rollout status deployment/frontend -n istio-traffic
```

Check the Pods and confirm that the application Pods have an `istio-proxy` sidecar:

```bash
kubectl get pods -n istio-traffic
```

---

## 4. Configure DestinationRule

Before routing traffic to named subsets, define those subsets using a `DestinationRule`.

Create `destination-rule.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: istio-traffic
spec:
  host: backend.istio-traffic.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

Apply it:

```bash
kubectl apply -f destination-rule.yaml
kubectl get destinationrule -n istio-traffic
```

### Explanation

- `host`: The Kubernetes Service to which the rule applies.
- `subsets`: Named groups of service workloads.
- `labels`: The Pod labels used to identify each subset.

The `v1` subset selects Pods labelled `version: v1`, while `v2` selects Pods labelled `version: v2`.

**Important:** A DestinationRule does not deploy a new version or create Pods. The Deployments create the Pods, and the subset labels identify them for traffic management.

---

## 5. Configure VirtualService

Create `virtual-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend
  namespace: istio-traffic
spec:
  hosts:
    - backend.istio-traffic.svc.cluster.local
  http:
    - route:
        - destination:
            host: backend.istio-traffic.svc.cluster.local
            subset: v1
          weight: 100
        - destination:
            host: backend.istio-traffic.svc.cluster.local
            subset: v2
          weight: 0
```

Apply the configuration:

```bash
kubectl apply -f virtual-service.yaml
kubectl get virtualservice -n istio-traffic
```

### Explanation

- `hosts`: The hostname to which the routing rules apply.
- `route`: The list of possible destinations.
- `subset`: The version defined in the DestinationRule.
- `weight`: The relative percentage of traffic assigned to a destination.

In this configuration, all traffic is directed to `backend-v1`.

### Test the routing

Run:

```bash
kubectl exec -n istio-traffic deployment/frontend -c curl -- \
  curl -sS http://backend.istio-traffic.svc.cluster.local:5678
```

Expected output:

```text
Response from backend-v1
```

---

## 6. Lab: Canary Traffic Splitting

A canary deployment introduces a new application version to a small percentage of traffic before gradually increasing its exposure.

For example:

- 90% of requests go to `backend-v1`.
- 10% of requests go to `backend-v2`.

Update the `http` section of `virtual-service.yaml`:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v1
        weight: 90
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v2
        weight: 10
```

Apply the updated configuration:

```bash
kubectl apply -f virtual-service.yaml
```

Send multiple requests:

```bash
for i in $(seq 1 30); do
  kubectl exec -n istio-traffic deployment/frontend -c curl -- \
    curl -sS http://backend.istio-traffic.svc.cluster.local:5678
done
```

You should see responses from both versions over repeated requests.

**Note:** The 90/10 weights describe the intended distribution of eligible traffic over time, not an exact ratio for every small batch of requests. Actual observations can vary.

This technique is useful for controlled releases, canary deployments, and testing new application versions.

---

## 7. Lab: Route Traffic Using HTTP Headers

Istio can route requests to a specific version when an HTTP header matches a condition.

For example, route requests containing `x-version: v2` to the second version, while sending all other requests to version 1.

Replace the `http` section of the VirtualService with:

```yaml
http:
  - match:
      - headers:
          x-version:
            exact: v2
    route:
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v2
  - route:
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v1
```

Apply the configuration:

```bash
kubectl apply -f virtual-service.yaml
```

Test the default route:

```bash
kubectl exec -n istio-traffic deployment/frontend -c curl -- \
  curl -sS http://backend.istio-traffic.svc.cluster.local:5678
```

Expected output:

```text
Response from backend-v1
```

Test the header-based route:

```bash
kubectl exec -n istio-traffic deployment/frontend -c curl -- \
  curl -sS -H "x-version: v2" \
  http://backend.istio-traffic.svc.cluster.local:5678
```

Expected output:

```text
Response from backend-v2
```

### Key point

Istio evaluates routing rules in order. Put specific matching rules before the general fallback rule.

Header-based routing can support testing, controlled access to new versions, and application experiments. However, a client-controlled header should not be treated as an authentication or authorization mechanism.

---

## 8. Lab: Configure Request Timeouts

A timeout defines how long a proxy should wait for a response before treating a request as failed.

Add a timeout to the route:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v1
    timeout: 3s
```

Apply the updated VirtualService:

```bash
kubectl apply -f virtual-service.yaml
```

In this example, Istio allows up to three seconds for the HTTP request handled by this route.

Timeouts are useful when a downstream service becomes slow or unresponsive.

**Best practice:** Choose timeouts based on application latency requirements and ensure that downstream timeouts, application timeouts, and upstream timeouts work together.

---

## 9. Lab: Configure Retries

A retry policy tells Istio to retry eligible failed HTTP requests.

Example:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-traffic.svc.cluster.local
          subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,unavailable
```

Apply the updated configuration:

```bash
kubectl apply -f virtual-service.yaml
```

### Explanation

- `attempts`: Maximum number of retry attempts, excluding the original request.
- `perTryTimeout`: Maximum time allowed for each attempt.
- `retryOn`: The failure conditions that trigger a retry.

Retries can help recover from temporary network errors or transient service failures.

**Important:** Retries can increase load on an unhealthy service. Configure them carefully, and consider idempotency before retrying operations that modify data. A retry does not guarantee that a failed request will succeed.

---

## 10. Load Balancing and Circuit Breaking

### Load balancing

Load balancing controls how traffic is distributed among eligible service endpoints.

A DestinationRule can specify a load-balancing policy:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: istio-traffic
spec:
  host: backend.istio-traffic.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

Apply it:

```bash
kubectl apply -f destination-rule.yaml
```

Common load-balancing options include:

- `ROUND_ROBIN`: Distributes requests across endpoints in rotation.
- `LEAST_REQUEST`: Prefers endpoints with fewer active requests.
- `RANDOM`: Selects an endpoint randomly.

### Circuit breaking

Circuit breaking limits connections or requests to help prevent an overloaded service from being overwhelmed.

Example DestinationRule:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: istio-traffic
spec:
  host: backend.istio-traffic.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

This example sets connection/request limits and configures outlier detection, which can temporarily eject unhealthy endpoints from the load-balancing pool.

These settings are illustrative. Tune them based on your workload and test their effect before using them in production.

---

## 11. Validate Traffic Management Configuration

Use the following commands to inspect your resources:

```bash
kubectl get virtualservice,destinationrule -n istio-traffic
```

Check the current routing configuration:

```bash
kubectl get virtualservice backend -n istio-traffic -o yaml
kubectl get destinationrule backend -n istio-traffic -o yaml
```

Check whether Istio proxies are synchronized:

```bash
istioctl proxy-status
```

Analyze configuration issues:

```bash
istioctl analyze -n istio-traffic
```

Inspect the proxy's clusters:

```bash
istioctl proxy-config clusters deployment/frontend.istio-traffic
```

Inspect the proxy's endpoints:

```bash
istioctl proxy-config endpoints deployment/frontend.istio-traffic
```

If traffic does not reach the expected version, verify:

1. The VirtualService hostname matches the intended service.
2. The DestinationRule hostname matches that service.
3. The subset labels match the actual Pod labels.
4. The service has ready endpoints.
5. The workload has an injected Envoy proxy.
6. The routing rules are applied in the correct order.

---

## 12. Common Troubleshooting Scenarios

| Problem | Possible cause | What to check |
|---|---|---|
| Requests always reach v1 | The VirtualService still routes 100% to v1 | Inspect route weights |
| Requests fail after creating a subset | The subset labels do not match Pod labels | Inspect DestinationRule and Pod labels |
| VirtualService changes have no effect | Hostname mismatch or invalid configuration | Run `istioctl analyze` |
| Requests return HTTP 503 | No healthy destination, routing issue, or circuit-breaking limits | Check endpoints, proxy status, and Envoy logs |
| Header routing fails | Header name/value mismatch or wrong rule order | Inspect the request and VirtualService |
| Retry policy does not help | Failure is not retryable or the deadline expires first | Check `retryOn`, timeouts, and application logs |

Useful commands:

```bash
kubectl get pods -n istio-traffic --show-labels
kubectl get endpointslices -n istio-traffic
kubectl logs deployment/frontend -n istio-traffic -c istio-proxy
kubectl logs deployment/backend-v1 -n istio-traffic -c istio-proxy
```

---

## 13. Cleanup

Remove the lab namespace and its resources:

```bash
kubectl delete namespace istio-traffic
```

This deletes the Deployments, Services, and Istio resources created inside that namespace.

---

## 14. Key Takeaways

- Istio traffic management controls how requests move between services.
- VirtualService defines routing rules.
- DestinationRule defines subsets and destination-level traffic policies.
- Gateway configures edge traffic handling.
- ServiceEntry registers external destinations.
- Weighted routing supports canary deployments.
- Header-based routing enables targeted request routing.
- Timeouts and retries help manage transient failures.
- Circuit breaking and outlier detection help limit the impact of unhealthy services.
- Correct hostnames, labels, and rule ordering are essential for routing to work.

## 15. Interview Questions

1. What is traffic management in Istio?
2. What is the difference between VirtualService and DestinationRule?
3. How does Istio perform canary deployments?
4. How do you route traffic based on HTTP headers?
5. What is the difference between a Kubernetes Service and an Istio VirtualService?
6. What is a service subset?
7. How do timeouts and retries work in Istio?
8. What is circuit breaking?
9. What is the purpose of an Istio Gateway?
10. How would you troubleshoot an HTTP 503 error in an Istio service mesh?

## 16. Official Documentation

- [Istio Traffic Management Concepts](https://istio.io/latest/docs/concepts/traffic-management/)
- [VirtualService Reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [DestinationRule Reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/)
- [Gateway Reference](https://istio.io/latest/docs/reference/config/networking/gateway/)
- [Request Timeouts](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)
- [Fault Injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)

## ➡️ Next Module

Continue to [Module 06: Canary Deployments](../06-canary-deployments/README.md) to learn how to roll out new application versions gradually using Istio traffic weights.
