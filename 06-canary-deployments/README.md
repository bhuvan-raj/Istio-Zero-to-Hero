# Module 06: Canary Deployments with Istio

## 📌 Overview

A **canary deployment** is a deployment strategy in which a new application version is released to a small percentage of traffic while the existing stable version continues serving most users.

The new version is monitored for errors, latency, and other issues. If it performs well, its traffic share is gradually increased until it serves all traffic. If problems occur, traffic can be shifted back to the stable version.

Istio makes canary deployments possible through **VirtualService traffic weights** and **DestinationRule subsets**, without requiring application code changes.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What a canary deployment is and why it is used
- Canary deployments versus rolling and blue-green deployments
- How VirtualService and DestinationRule work together
- How to deploy multiple application versions
- How to split traffic between stable and canary versions
- How to perform a gradual rollout and rollback
- How to validate traffic distribution
- How to troubleshoot common canary deployment problems

## 📚 Prerequisites

- A working Kubernetes cluster
- Istio installed and configured
- `kubectl` and `istioctl` installed
- Basic knowledge of Deployments and Services
- Completion of Module 04: Your First Istio Application
- Completion of Module 05: Traffic Management

---

## 1. What Is a Canary Deployment?

Suppose your application currently runs version `v1`. Your team has developed version `v2` with new features and bug fixes.

Deploying `v2` directly to every user could be risky. If the new version has a bug, many users could be affected.

Instead, you can introduce `v2` gradually:

1. Deploy `v2` alongside `v1`.
2. Send 5% of traffic to `v2`.
3. Monitor application health, errors, and latency.
4. Increase traffic to 10%, 25%, 50%, and eventually 100%.
5. Roll back the traffic if the new version causes problems.

### Architecture

```text
                   Users
                     |
                     v
               Istio Gateway
                     |
                     v
                VirtualService
                     |
              Traffic Distribution
                /            \
          90% traffic      10% traffic
              |                |
              v                v
         Stable v1          Canary v2
         Deployment         Deployment
              |                |
              v                v
          Envoy Proxy       Envoy Proxy
```

The percentages shown are illustrative. In this module, we will route requests through the Kubernetes Service hostname using Istio's sidecar proxies.

## 2. Why Use Canary Deployments?

- **Reduced risk:** Only a portion of traffic reaches the new version initially.
- **Early issue detection:** Problems can be identified before a full rollout.
- **Controlled rollout:** Traffic can be increased gradually.
- **Fast rollback:** Traffic can be redirected to the stable version.
- **Production validation:** The new version can be tested with real workloads.

### Canary vs. Other Deployment Strategies

| Strategy | How it works | Main benefit |
|---|---|---|
| Rolling deployment | Replaces old Pods with new Pods incrementally | Gradual replacement of instances |
| Blue-green deployment | Maintains two environments and switches traffic between them | Fast environment-level switching |
| Canary deployment | Gradually shifts a percentage of traffic to the new version | Limits exposure to a new release |

A Kubernetes rolling update and an Istio canary deployment are not the same thing. A rolling update changes the set of Pods behind a workload, while Istio traffic weights can direct requests between separately deployed versions.

---

## 3. Lab Architecture

In this lab, we will create:

- `backend-v1`: The stable version
- `backend-v2`: The canary version
- `backend`: A Kubernetes Service selecting both versions
- `DestinationRule`: Defines the `v1` and `v2` subsets
- `VirtualService`: Controls traffic distribution
- `frontend`: A client used to send test requests

```text
frontend
   |
   v
backend.istio-canary.svc.cluster.local
   |
   v
Istio VirtualService
   |
   +---- 90% ----> backend-v1
   |
   +---- 10% ----> backend-v2
```

Both backend versions use the same application image but return different response messages so that you can identify which version handled each request.

## 4. Create the Namespace

Create a namespace and enable automatic sidecar injection:

```bash
kubectl create namespace istio-canary

kubectl label namespace istio-canary \
  istio-injection=enabled
```

If the namespace already exists, run only the label command.

Verify the label:

```bash
kubectl get namespace istio-canary --show-labels
```

The namespace should show `istio-injection=enabled`.

## 5. Deploy Stable and Canary Versions

Create a file named `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v1
  namespace: istio-canary
spec:
  replicas: 3
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
            - "-text=Stable version v1"
          ports:
            - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-v2
  namespace: istio-canary
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
            - "-text=Canary version v2"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-canary
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
  namespace: istio-canary
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

Wait for the Deployments to become ready:

```bash
kubectl rollout status deployment/backend-v1 -n istio-canary
kubectl rollout status deployment/backend-v2 -n istio-canary
kubectl rollout status deployment/frontend -n istio-canary
```

Check the Pods:

```bash
kubectl get pods -n istio-canary
```

Confirm that the application Pods have an `istio-proxy` sidecar:

```bash
kubectl get pods -n istio-canary \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.containers[*].name}{"\n"}{end}'
```

## 6. Create DestinationRule Subsets

A `DestinationRule` defines named subsets based on workload labels. These subsets allow Istio to distinguish between stable and canary versions.

Create `destination-rule.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: istio-canary
spec:
  host: backend.istio-canary.svc.cluster.local
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
```

Verify:

```bash
kubectl get destinationrule -n istio-canary
kubectl describe destinationrule backend -n istio-canary
```

### How it works

- `host`: Identifies the backend Kubernetes Service.
- `subsets`: Defines the available application versions.
- `labels`: Selects Pods belonging to each version.

The `v1` subset selects Pods labelled `version: v1`, while the `v2` subset selects Pods labelled `version: v2`.

**Important:** The Deployment labels must match the subset labels. Otherwise, Istio may have no endpoints for the selected subset.

## 7. Configure the Initial Canary Split

Initially, we want all traffic to reach the stable version.

Create `virtual-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: backend
  namespace: istio-canary
spec:
  hosts:
    - backend.istio-canary.svc.cluster.local
  http:
    - route:
        - destination:
            host: backend.istio-canary.svc.cluster.local
            subset: v1
          weight: 100
        - destination:
            host: backend.istio-canary.svc.cluster.local
            subset: v2
          weight: 0
```

Apply the configuration:

```bash
kubectl apply -f virtual-service.yaml
```

At this stage, the intended distribution is:

| Version | Traffic weight |
|---|---:|
| Stable v1 | 100% |
| Canary v2 | 0% |

### Test the stable version

```bash
kubectl exec -n istio-canary deployment/frontend -c curl -- \
  curl -sS http://backend.istio-canary.svc.cluster.local:5678
```

Expected output:

```text
Stable version v1
```

---

## 8. Shift 10% of Traffic to the Canary

Now update the `http` section of `virtual-service.yaml`:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v1
        weight: 90
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v2
        weight: 10
```

Apply the change:

```bash
kubectl apply -f virtual-service.yaml
```

The intended distribution is now:

| Version | Traffic weight |
|---|---:|
| Stable v1 | 90% |
| Canary v2 | 10% |

Send repeated requests:

```bash
for i in $(seq 1 50); do
  kubectl exec -n istio-canary deployment/frontend -c curl -- \
    curl -sS http://backend.istio-canary.svc.cluster.local:5678
done
```

You should see responses from both versions. With enough requests, the observed distribution should broadly approach the configured weights.

**Note:** Traffic weights are probabilistic over requests, not a guarantee that exactly 5 out of every 50 requests will reach the canary. Small samples can differ significantly from the configured percentages.

## 9. Gradually Increase Canary Traffic

As you gain confidence in the new version, increase its traffic share.

### Stage 1: 10% canary

```yaml
weight: 90  # v1
weight: 10  # v2
```

### Stage 2: 25% canary

```yaml
weight: 75  # v1
weight: 25  # v2
```

### Stage 3: 50% canary

```yaml
weight: 50  # v1
weight: 50  # v2
```

### Stage 4: 100% canary

```yaml
weight: 0    # v1
weight: 100  # v2
```

For each stage:

1. Update the weights in `virtual-service.yaml`.
2. Apply the configuration.
3. Send test requests.
4. Check application logs, error rates, and latency.
5. Continue only if the canary meets your release criteria.

For real production rollouts, use meaningful health indicators and sufficient traffic before increasing the percentage. A few successful requests alone do not prove that a release is healthy.

## 10. Roll Back a Canary Deployment

If the new version has errors, route all traffic back to the stable version.

Update the VirtualService weights:

```yaml
http:
  - route:
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v1
        weight: 100
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v2
        weight: 0
```

Apply the rollback:

```bash
kubectl apply -f virtual-service.yaml
```

Verify the result:

```bash
kubectl get virtualservice backend -n istio-canary -o yaml
```

Test the application again:

```bash
kubectl exec -n istio-canary deployment/frontend -c curl -- \
  curl -sS http://backend.istio-canary.svc.cluster.local:5678
```

Expected output:

```text
Stable version v1
```

This changes traffic routing without deleting the canary Deployment.

**Important:** A traffic rollback does not undo database migrations, external side effects, or incompatible data changes made by the new version. Plan application and database compatibility separately.

## 11. Canary Deployment with Header-Based Routing

Sometimes you want only selected testers to access the canary while normal users continue using the stable version.

Istio can route requests based on HTTP headers.

For example, requests containing `x-canary: true` will go to version 2. All other requests will go to version 1.

Update the `http` section of the VirtualService:

```yaml
http:
  - match:
      - headers:
          x-canary:
            exact: "true"
    route:
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v2
  - route:
      - destination:
          host: backend.istio-canary.svc.cluster.local
          subset: v1
```

Apply it:

```bash
kubectl apply -f virtual-service.yaml
```

Test the default route:

```bash
kubectl exec -n istio-canary deployment/frontend -c curl -- \
  curl -sS http://backend.istio-canary.svc.cluster.local:5678
```

Expected output:

```text
Stable version v1
```

Test the canary route:

```bash
kubectl exec -n istio-canary deployment/frontend -c curl -- \
  curl -sS -H "x-canary: true" \
  http://backend.istio-canary.svc.cluster.local:5678
```

Expected output:

```text
Canary version v2
```

The header match must appear before the general fallback route.

**Security note:** Header-based routing is useful for testing, but a client-controlled header is not proof of user identity. Protect sensitive preview environments with proper authentication and authorization.

## 12. Monitor the Canary Deployment

A successful canary rollout needs observability.

Monitor the following indicators:

| Metric | Why it matters |
|---|---|
| Request success rate | Detects failures introduced by the new version |
| HTTP 5xx error rate | Identifies server-side errors |
| Request latency | Detects performance regressions |
| CPU and memory usage | Reveals resource pressure |
| Application logs | Helps identify exceptions and errors |
| Request volume | Provides context for interpreting error percentages |

### Useful commands

Check the Deployments:

```bash
kubectl get deployments -n istio-canary
```

Check the Pods and labels:

```bash
kubectl get pods -n istio-canary --show-labels
```

Inspect the VirtualService:

```bash
kubectl describe virtualservice backend -n istio-canary
```

Inspect the DestinationRule:

```bash
kubectl describe destinationrule backend -n istio-canary
```

Check Istio proxy synchronization:

```bash
istioctl proxy-status
```

Analyze configuration:

```bash
istioctl analyze -n istio-canary
```

If Prometheus and Grafana are installed, use Istio telemetry to compare request volume, error rates, and latency between the stable and canary workloads. The exact dashboard and query depend on your telemetry setup and labels.

## 13. Troubleshooting

| Problem | Possible cause | What to check |
|---|---|---|
| Only v1 receives traffic | Canary weight is zero or the VirtualService was not updated | Inspect the VirtualService |
| Canary requests fail | Incorrect subset labels or no ready endpoints | Check Pod labels and endpoints |
| Traffic percentages appear incorrect | Too few requests or the test method adds overhead | Send more requests and inspect telemetry |
| VirtualService changes have no effect | Hostname mismatch or proxy configuration issue | Run `istioctl analyze` and check proxy status |
| HTTP 503 responses | No healthy destination or a routing/policy problem | Check endpoints, Envoy logs, and DestinationRule settings |
| Header routing does not work | Header mismatch or rule ordering problem | Verify the request header and match rules |
| Canary is healthy but users experience errors | Production traffic differs from test traffic | Check latency, dependencies, logs, and telemetry |

Check the service endpoints:

```bash
kubectl get endpointslices -n istio-canary
```

Inspect proxy logs:

```bash
kubectl logs deployment/frontend -n istio-canary -c istio-proxy
```

Inspect the backend logs:

```bash
kubectl logs deployment/backend-v2 -n istio-canary -c app
```

---

## 14. Best Practices

- Keep stable and canary versions in separate Deployments.
- Use distinct version labels, such as `version: v1` and `version: v2`.
- Define DestinationRule subsets that match those labels.
- Start with a small traffic percentage and increase it gradually.
- Monitor latency, error rates, and resource usage at every stage.
- Automate rollback when release health criteria fail.
- Use compatible database schemas and API contracts during rollout.
- Avoid aggressive retries that could overload an unhealthy service.
- Keep routing configuration in version control and review changes.
- Use authentication and authorization for restricted canary access.
- Test the rollback procedure before relying on it in production.

## 15. Hands-On Exercises

Try completing these tasks independently:

- [ ] Deploy stable and canary versions of a backend service.
- [ ] Configure a DestinationRule with `v1` and `v2` subsets.
- [ ] Route 100% of traffic to `v1`.
- [ ] Shift 10% of traffic to `v2`.
- [ ] Increase the canary traffic to 25%, 50%, and 100%.
- [ ] Roll back to the stable version.
- [ ] Route requests to `v2` using a custom HTTP header.
- [ ] Inspect the Envoy proxy status and application logs.
- [ ] Explain why a 10% traffic weight does not guarantee exactly 10% in every small batch.

## 16. Interview Questions

1. What is a canary deployment?
2. How does Istio implement canary deployments?
3. What is the role of a VirtualService in traffic splitting?
4. Why do we need a DestinationRule for version-based routing?
5. What is the difference between a Kubernetes rolling update and an Istio canary deployment?
6. How do you route 10% of traffic to a new application version?
7. How do you roll back a canary deployment without deleting the new Deployment?
8. How would you identify a canary version that causes increased HTTP 5xx errors?
9. How does header-based routing work in Istio?
10. What risks remain after routing all traffic away from a failed canary?

## 17. Key Takeaways

- Canary deployments reduce release risk by exposing a new version to a limited share of traffic.
- Istio uses VirtualService traffic weights to distribute requests.
- DestinationRule subsets identify the stable and canary workloads.
- Traffic can be shifted gradually or redirected immediately during rollback.
- Header-based routing supports targeted testing.
- Monitoring is essential to decide whether a canary should progress or roll back.
- Traffic rollback does not reverse application-side effects or database changes.

## 📖 Official Documentation

- [Istio Traffic Management Concepts](https://istio.io/latest/docs/concepts/traffic-management/)
- [VirtualService Reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [DestinationRule Reference](https://istio.io/latest/docs/reference/config/networking/destination-rule/)
- [Istio Observability](https://istio.io/latest/docs/concepts/observability/)

## ➡️ Next Module

Continue to [Module 07: Resilience and Fault Tolerance](../07-resilience-fault-tolerance/README.md) to learn about timeouts, retries, circuit breaking, and fault injection.
