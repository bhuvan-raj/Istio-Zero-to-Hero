# Module 07: Resilience and Fault Tolerance in Istio

## 📌 Overview

In a microservices architecture, services depend on one another to complete requests. If one service becomes slow, unavailable, or overloaded, the failure can affect other services and potentially bring down the entire application.

**Resilience** is the ability of an application to continue operating or recover gracefully when failures occur.

Istio provides traffic-management features that help improve application resilience without requiring changes to application code. These features are configured through resources such as `VirtualService` and `DestinationRule` and enforced by Envoy proxies.

In this module, you will learn how to configure timeouts, retries, circuit breaking, outlier detection, and fault injection.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What resilience and fault tolerance mean
- How timeouts prevent requests from waiting indefinitely
- How retries help recover from transient failures
- How circuit breaking protects overloaded services
- How outlier detection temporarily removes unhealthy endpoints
- How to inject artificial delays and HTTP errors
- How to test failure scenarios and troubleshoot HTTP 503 errors
- How to design resilient microservice communication

## 📚 Prerequisites

- A working Kubernetes cluster
- Istio installed
- `kubectl` and `istioctl` installed
- Basic knowledge of Deployments, Services, VirtualServices, and DestinationRules
- Completion of Modules 04, 05, and 06

---

## 1. What Is Resilience?

Consider an application with three microservices:

```text
Frontend
   |
   v
Order Service
   |
   v
Payment Service
   |
   v
Database
```

Suppose the Payment Service becomes slow. The Order Service may continue waiting for a response. If many requests accumulate, the Order Service can exhaust its available connections or resources, affecting the Frontend as well.

This is an example of a **cascading failure**.

Resilience mechanisms help limit the impact of these failures.

### Important resilience mechanisms

| Mechanism | Purpose |
|---|---|
| Timeout | Stops waiting after a defined duration |
| Retry | Attempts a failed request again when appropriate |
| Circuit breaking | Limits connections and requests to a destination |
| Outlier detection | Temporarily ejects unhealthy endpoints from a load-balancing pool |
| Fault injection | Introduces artificial failures to test application behavior |
| Fallback handling | Provides an alternative response when a dependency fails |

Istio helps manage communication failures, but your application must still handle errors appropriately. A proxy cannot automatically guarantee that a business operation succeeds or that a fallback response is correct.

---

## 2. How Istio Provides Resilience

Istio uses Envoy proxies to apply traffic-management policies to requests flowing through the service mesh.

```text
Frontend Application
        |
        v
   Envoy Proxy
        |
        |  Timeout / Retry / Routing
        |  Connection Limits
        v
   Envoy Proxy
        |
        v
Backend Application
```

The control plane distributes configuration to Envoy proxies. The proxies enforce the configured policies when handling traffic.

For example, if the backend takes too long to respond, the calling proxy can stop waiting after the configured timeout. Depending on the retry policy and remaining time budget, it may retry the request.

**Important:** Istio's resilience settings do not replace application-level timeouts, database transaction handling, idempotency, or error handling. These mechanisms must be designed to work together.

---

## 3. Lab Setup

We will create a backend service and a frontend client. The frontend will send requests to the backend while we apply different resilience policies.

### Architecture

```text
frontend
   |
   v
Istio Envoy Proxy
   |
   v
VirtualService
   |
   v
backend Service
   |
   v
backend Pods
```

Create a namespace and enable sidecar injection:

```bash
kubectl create namespace istio-resilience

kubectl label namespace istio-resilience \
  istio-injection=enabled
```

If the namespace already exists, run the label command directly.

### Create the application

Create `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: istio-resilience
spec:
  replicas: 2
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
            - "-text=Backend response successful"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-resilience
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
  namespace: istio-resilience
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
```

Wait for the Deployments:

```bash
kubectl rollout status deployment/backend -n istio-resilience
kubectl rollout status deployment/frontend -n istio-resilience
```

Check the Pods:

```bash
kubectl get pods -n istio-resilience
```

Confirm that the frontend and backend Pods have an `istio-proxy` sidecar.

### Test normal communication

```bash
kubectl exec -n istio-resilience deployment/frontend -c curl -- \
  curl -sS http://backend.istio-resilience.svc.cluster.local:5678
```

Expected output:

```text
Backend response successful
```

This confirms that the client can communicate with the backend before resilience policies are applied.

---

## 4. Timeouts

### What is a timeout?

A timeout defines how long a client or proxy waits for a response before considering a request unsuccessful.

Without a suitable timeout, a request can wait too long for a slow or unresponsive dependency, increasing latency and consuming resources.

For HTTP requests, Istio allows you to configure a timeout in a `VirtualService`.

### Example

Create `virtual-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend
  namespace: istio-resilience
spec:
  hosts:
    - backend.istio-resilience.svc.cluster.local
  http:
    - route:
        - destination:
            host: backend.istio-resilience.svc.cluster.local
            port:
              number: 5678
      timeout: 3s
```

Apply the configuration:

```bash
kubectl apply -f virtual-service.yaml
```

The proxy will enforce a three-second timeout for requests handled by this route.

### Why use timeouts?

- Prevent requests from waiting indefinitely.
- Limit the time spent waiting for slow dependencies.
- Help control end-to-end request latency.
- Reduce resource consumption caused by requests that take too long.

**Best practice:** Set timeouts according to your application's latency requirements. A timeout that is too short can reject legitimate requests, while a timeout that is too long can worsen resource exhaustion.

---

## 5. Retries

### What is a retry?

A retry is another attempt to perform a request after an eligible failure.

For example, a request may fail because of a temporary connection problem. A retry can allow it to succeed if the problem clears quickly.

Create or update the `http` section of `virtual-service.yaml`:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-resilience.svc.cluster.local
          port:
            number: 5678
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,unavailable
```

Apply it:

```bash
kubectl apply -f virtual-service.yaml
```

### Explanation

- `attempts`: The maximum number of retry attempts, excluding the original request.
- `perTryTimeout`: The maximum time allowed for each attempt.
- `retryOn`: The failure conditions that trigger a retry.

Retries can improve reliability when failures are temporary, but they do not guarantee that a request will eventually succeed.

### Important considerations

1. Retries can increase traffic to an already unhealthy service.
2. Repeated attempts can increase request latency.
3. The overall request timeout and per-attempt timeout should be planned together.
4. Avoid blindly retrying operations that modify data unless they are designed to be safely repeated.

For example, retrying a read-only request is usually less risky than retrying a payment operation that might already have been processed.

---

## 6. Circuit Breaking

### What is circuit breaking?

Circuit breaking limits the number of connections, pending requests, or active requests sent to a destination.

When configured limits are reached, Envoy can reject additional requests instead of allowing an overloaded service to receive an unlimited amount of work.

Think of an electrical circuit breaker: when the system detects an unsafe condition, the circuit interrupts the flow to protect the system.

In Istio, circuit-breaking settings are configured using a `DestinationRule`.

### Create a DestinationRule

Create `destination-rule.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: istio-resilience
spec:
  host: backend.istio-resilience.svc.cluster.local
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
      maxEjectionPercent: 50
```

Apply the configuration:

```bash
kubectl apply -f destination-rule.yaml
```

### Explanation

| Setting | Meaning |
|---|---|
| `maxConnections` | Limits connections in the configured connection pool |
| `http1MaxPendingRequests` | Limits pending HTTP/1.1 requests waiting for a connection |
| `http2MaxRequests` | Limits concurrent outstanding HTTP/2 requests |
| `consecutive5xxErrors` | Configures the consecutive server-error threshold used for outlier detection |
| `interval` | Sets the interval for outlier detection analysis |
| `baseEjectionTime` | Sets the base duration for which an unhealthy host is ejected |
| `maxEjectionPercent` | Limits the percentage of hosts that can be ejected |

These are illustrative values, not universal production defaults. Tune them according to the workload, protocol, replica count, and expected traffic.

**Important distinction:** Connection-pool limits and outlier detection are related resilience features but are not the same thing. Connection-pool limits restrict traffic capacity. Outlier detection identifies unhealthy endpoints and temporarily removes them from the load-balancing pool.

### How circuit breaking helps

Suppose the backend is overloaded and can safely handle only a limited amount of concurrent work. Circuit-breaking limits help prevent excess requests from accumulating and contributing to a cascading failure.

When a request is rejected by a proxy, the caller may receive an HTTP 503 response. This does not necessarily mean the backend application itself returned a 503.

---

## 7. Outlier Detection

### What is outlier detection?

Outlier detection identifies endpoints that appear unhealthy based on configured failure conditions and temporarily ejects them from the load-balancing pool.

Suppose a Service has three backend Pods:

```text
backend Service
      |
      +---- Pod A: Healthy
      |
      +---- Pod B: Unhealthy
      |
      +---- Pod C: Healthy
```

If Pod B repeatedly returns server errors and meets the configured ejection criteria, Envoy can temporarily stop sending traffic to it while continuing to use eligible healthy endpoints.

### Important settings

- `consecutive5xxErrors`: Consecutive HTTP 5xx responses used as an unhealthy-host signal.
- `interval`: How often the outlier detection analysis runs.
- `baseEjectionTime`: The base ejection duration.
- `maxEjectionPercent`: The maximum percentage of hosts that can be ejected.

Outlier detection does not repair a failed application. It reduces traffic to endpoints identified as unhealthy, subject to the configured policy and the number of eligible endpoints.

If all endpoints are unhealthy or ejected, requests can still fail.

---

## 8. Fault Injection

### What is fault injection?

Fault injection deliberately introduces failures into a system to test how the application behaves when dependencies are slow or unavailable.

Instead of waiting for a real production failure, you simulate one in a controlled environment.

Istio supports two common HTTP fault-injection mechanisms:

- **Delay:** Adds artificial latency to matching requests.
- **Abort:** Returns a configured HTTP error for matching requests.

Use fault injection in a test or staging environment, or carefully scope it to specific requests. Avoid applying test faults broadly to production traffic.

### Lab A: Inject an HTTP delay

Replace the HTTP routing section of `virtual-service.yaml` with:

```yaml
http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 5s
    route:
      - destination:
          host: backend.istio-resilience.svc.cluster.local
          port:
            number: 5678
```

Apply it:

```bash
kubectl apply -f virtual-service.yaml
```

Send a request and measure the response time:

```bash
time kubectl exec -n istio-resilience deployment/frontend -c curl -- \
  curl -sS --max-time 10 \
  http://backend.istio-resilience.svc.cluster.local:5678
```

The request should experience approximately five seconds of additional delay, plus normal network and command-execution overhead.

### What did we learn?

The backend can be healthy and still respond slowly from the client's perspective. Delays in dependencies can increase latency across the entire request path.

This lab demonstrates why timeout budgets and downstream latency matter.

### Lab B: Inject an HTTP abort

Replace the HTTP routing section with:

```yaml
http:
  - fault:
      abort:
        percentage:
          value: 100
        httpStatus: 503
    route:
      - destination:
          host: backend.istio-resilience.svc.cluster.local
          port:
            number: 5678
```

Apply it:

```bash
kubectl apply -f virtual-service.yaml
```

Send a request:

```bash
kubectl exec -n istio-resilience deployment/frontend -c curl -- \
  curl -i http://backend.istio-resilience.svc.cluster.local:5678
```

You should receive an HTTP 503 response for the matching request.

The abort is injected by Istio's traffic-handling layer; it does not necessarily mean the backend application is returning HTTP 503 itself.

### Important configuration note

For these labs, use the delay or abort rule as the active fault-injection configuration. Do not combine fault injection with retry or timeout configuration in the same VirtualService HTTP route; Istio documents restrictions on combining these features.

To test a timeout or retry policy, restore the corresponding timeout/retry route configuration before running that test.

---

## 9. Compare the Resilience Mechanisms

| Mechanism | What it does | Example use case |
|---|---|---|
| Timeout | Stops waiting after a deadline | A dependency is taking too long |
| Retry | Reattempts eligible failed requests | A temporary network failure |
| Circuit breaking | Limits connections or requests | A service is overloaded |
| Outlier detection | Temporarily ejects unhealthy endpoints | One replica repeatedly returns errors |
| Fault injection | Deliberately simulates failures | Testing error handling before release |
| Fallback | Provides an alternative response | Showing cached or default data when a dependency fails |

These mechanisms complement one another, but they solve different problems. No single mechanism can guarantee that a distributed application will remain available under every failure scenario.

---

## 10. Troubleshooting

| Problem | Possible cause | What to check |
|---|---|---|
| Timeout does not occur | Request is not matching the VirtualService host or route | Inspect the VirtualService and hostname |
| Retries do not happen | Failure condition is not covered by the retry policy | Review `retryOn` and proxy logs |
| HTTP 503 appears | Fault injection, circuit-breaking limits, no healthy endpoints, or upstream failure | Inspect policies, endpoints, and proxy logs |
| Delay is not visible | Fault rule is not applied to the tested traffic | Check host matching and proxy synchronization |
| All requests fail after ejection | No healthy endpoints remain | Check Pods, readiness, and ejection policy |
| Application still fails despite retries | Failure is persistent or the overall deadline is exhausted | Check application errors, timeout budgets, and dependency health |
| DestinationRule appears ineffective | Hostname mismatch or configuration not applied | Run `istioctl analyze` and inspect proxy status |

Useful commands:

```bash
kubectl get virtualservice,destinationrule -n istio-resilience
kubectl get pods -n istio-resilience -o wide
kubectl get endpointslices -n istio-resilience
istioctl proxy-status
istioctl analyze -n istio-resilience
```

Inspect proxy logs:

```bash
kubectl logs deployment/frontend -n istio-resilience -c istio-proxy
```

Inspect the applied routing policy:

```bash
kubectl get virtualservice backend -n istio-resilience -o yaml
kubectl get destinationrule backend -n istio-resilience -o yaml
```

---

## 11. Best Practices

- Set realistic timeouts based on service-level objectives.
- Design retry policies with an overall latency budget.
- Retry only failures that are appropriate to retry.
- Make write operations idempotent or use another safe deduplication mechanism where retries are possible.
- Avoid excessive retries that amplify load during an outage.
- Configure connection-pool limits based on workload capacity.
- Use outlier detection carefully when the number of replicas is small.
- Test fault injection in a controlled environment.
- Monitor latency, error rates, request volume, and resource usage.
- Ensure application code handles errors returned by Envoy.
- Test how multiple resilience mechanisms interact before deploying them to production.
- Remember that service-mesh resilience cannot fix application bugs, database corruption, or incompatible data migrations.

---

## 12. Hands-On Exercises

- [ ] Deploy the frontend and backend workloads.
- [ ] Configure a three-second request timeout.
- [ ] Configure a retry policy for eligible failures.
- [ ] Apply connection-pool limits using a DestinationRule.
- [ ] Configure outlier detection.
- [ ] Inject a five-second HTTP delay.
- [ ] Inject an HTTP 503 abort.
- [ ] Compare the results of normal requests and fault-injected requests.
- [ ] Inspect Envoy logs and Istio configuration.
- [ ] Explain the difference between circuit breaking and outlier detection.
- [ ] Restore the normal routing configuration after testing.

## 13. Interview Questions

1. What is resilience in a microservices architecture?
2. What is the difference between a timeout and a retry?
3. How does Istio implement circuit breaking?
4. What is outlier detection?
5. What is the difference between circuit breaking and outlier detection?
6. What is fault injection, and why is it useful?
7. How can retries increase the load on an unhealthy service?
8. What causes HTTP 503 responses in an Istio service mesh?
9. How do you test whether an application can tolerate a slow downstream dependency?
10. Why should retry policies consider idempotency?
11. What happens when all endpoints are unhealthy?
12. Does Istio resilience eliminate the need for application-level error handling?

## 14. Key Takeaways

- Resilience helps applications tolerate failures and recover gracefully.
- Timeouts bound how long a request waits for a response.
- Retries can recover from transient failures but may increase load.
- Circuit breaking limits connections and requests to protect services.
- Outlier detection temporarily removes unhealthy endpoints from a load-balancing pool.
- Fault injection lets you test failure-handling behavior deliberately.
- Envoy and Istio can manage network-level resilience, but application-level error handling remains essential.

## 📖 Official Documentation

- [Istio Traffic Management Concepts](https://istio.io/latest/docs/concepts/traffic-management/)
- [Request Timeouts](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)
- [Circuit Breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)
- [Fault Injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)
- [VirtualService Reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [DestinationRule Reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

## ➡️ Next Module

Continue to [Module 08: Istio Security and Zero Trust](../08-istio-security-zero-trust/README.md) to learn about workload identity, mutual TLS, authentication, and authorization policies.
