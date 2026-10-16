# Module 04: Your First Istio Application

In this module, you will deploy your first application in an Istio service mesh, enable Envoy sidecar injection, test communication between microservices, and inspect the traffic using Istio tools.

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [1. Understand the Application Architecture](#1-understand-the-application-architecture)
- [2. Create a Namespace](#2-create-a-namespace)
- [3. Enable Automatic Sidecar Injection](#3-enable-automatic-sidecar-injection)
- [4. Deploy the Sample Application](#4-deploy-the-sample-application)
- [5. Verify the Deployment](#5-verify-the-deployment)
- [6. Test Application Communication](#6-test-application-communication)
- [7. Inspect Envoy Sidecars](#7-inspect-envoy-sidecars)
- [8. Explore Istio Traffic Management](#8-explore-istio-traffic-management)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Clean Up](#10-clean-up)
- [Key Takeaways](#key-takeaways)
- [Interview Questions](#interview-questions)
- [Next Module](#next-module)
- [Official References](#official-references)

## Learning Objectives

By the end of this module, you will be able to:

- Deploy an application inside an Istio-enabled namespace.
- Understand how Envoy sidecars are injected into application Pods.
- Test communication between Kubernetes Services.
- Inspect Envoy proxy configuration.
- Understand how Istio participates in application traffic flow.
- Troubleshoot basic service-mesh communication issues.

## Prerequisites

- A working Kubernetes cluster.
- Istio installed using `istioctl`.
- `kubectl` and `istioctl` available in your terminal.
- Basic knowledge of Kubernetes Deployments, Services, and Pods.
- Completion of [Module 03: Installing Istio](../03-installing-istio/README.md).

Verify your cluster:

```bash
kubectl get nodes
istioctl version
```

## 1. Understand the Application Architecture

We will deploy two simple workloads:

- **Frontend:** A client Pod that sends HTTP requests.
- **Backend:** An HTTP server that responds to requests.

Each workload will have an Envoy sidecar injected by Istio.

### Architecture

```text
             Frontend Pod
        +---------------------+
        |  Client Container   |
        |         |           |
        |    Envoy Proxy      |
        +---------|-----------+
                  |
                  | HTTP request
                  v
          Kubernetes Service
                  |
                  v
             Backend Pod
        +---------------------+
        |  HTTP Server        |
        |         |           |
        |    Envoy Proxy      |
        +---------------------+
```

The frontend sends a request to the backend Kubernetes Service. Istio's data-plane proxies participate in handling the traffic according to the mesh configuration.

**Important:** A Kubernetes Service provides service discovery and load balancing. Istio adds capabilities such as traffic policies, telemetry, and workload-level security.

## 2. Create a Namespace

Create a dedicated namespace for the lab:

```bash
kubectl create namespace istio-demo
```

Verify it:

```bash
kubectl get namespaces
```

Using a separate namespace keeps the lab resources isolated from other applications.

## 3. Enable Automatic Sidecar Injection

Label the namespace so Istio can automatically inject Envoy sidecars into eligible new Pods:

```bash
kubectl label namespace istio-demo istio-injection=enabled
```

Verify the label:

```bash
kubectl get namespace istio-demo --show-labels
```

You should see:

```text
istio-injection=enabled
```

When a Pod is created in this namespace, the Istio injection webhook can modify its Pod specification to include an `istio-proxy` container.

Existing Pods are not automatically modified when the label is added. For this lab, we will create the workloads after enabling injection.

## 4. Deploy the Sample Application

We will use the `hashicorp/http-echo` container image to create a simple HTTP backend and a frontend client.

Create a file named `application.yaml` with the following content.

### application.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: istio-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Hello from the Istio backend!"
          ports:
            - name: http
              containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-demo
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
  namespace: istio-demo
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
          command:
            - "sh"
            - "-c"
            - "sleep 3650d"
```

Apply the configuration:

```bash
kubectl apply -f application.yaml
```

Check the Deployments and Services:

```bash
kubectl get deployments,services -n istio-demo
```

Wait for the Deployments to become ready:

```bash
kubectl rollout status deployment/backend -n istio-demo
kubectl rollout status deployment/frontend -n istio-demo
```

**What did we create?**

- A backend Deployment running an HTTP server.
- A ClusterIP Service exposing the backend on port `5678`.
- A frontend Deployment containing a `curl` client.
- Two workloads that should receive Envoy sidecars through automatic injection.

## 5. Verify the Deployment

List the Pods:

```bash
kubectl get pods -n istio-demo
```

Check the container names in each Pod:

```bash
kubectl get pods -n istio-demo \
  -o custom-columns='POD:.metadata.name,CONTAINERS:.spec.containers[*].name'
```

Expected container names should include:

- Backend: `http-echo` and `istio-proxy`
- Frontend: `curl` and `istio-proxy`

The precise Pod names will vary.

If `istio-proxy` is missing, revisit the namespace label and injection webhook configuration.

## 6. Test Application Communication

### Step 1: Find the frontend Pod

```bash
kubectl get pods -n istio-demo -l app=frontend
```

### Step 2: Set the frontend Pod name

```bash
FRONTEND_POD=$(kubectl get pod -n istio-demo \
  -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')
```

Verify the variable:

```bash
echo "$FRONTEND_POD"
```

### Step 3: Send a request to the backend

```bash
kubectl exec -n istio-demo "$FRONTEND_POD" -c curl -- \
  curl -sS http://backend.istio-demo.svc.cluster.local:5678
```

Expected response:

```text
Hello from the Istio backend!
```

This confirms that the frontend can reach the backend through its Kubernetes Service DNS name.

### Step 4: Test using the short Service name

Because the frontend and backend are in the same namespace, you can also use:

```bash
kubectl exec -n istio-demo "$FRONTEND_POD" -c curl -- \
  curl -sS http://backend:5678
```

Kubernetes DNS resolves `backend` to the backend Service in the current namespace.

### Step 5: Send multiple requests

```bash
kubectl exec -n istio-demo "$FRONTEND_POD" -c curl -- \
  sh -c 'i=1; while [ "$i" -le 5 ]; do curl -sS http://backend:5678; echo; i=$((i+1)); done'
```

You should receive the backend response repeatedly.

At this stage, we have verified basic application connectivity. We have not yet configured advanced Istio traffic policies.

## 7. Inspect Envoy Sidecars

Now we will inspect how Istio has configured the application workloads.

### Check proxy status

```bash
istioctl proxy-status
```

This shows the Envoy proxies known to Istiod and their configuration synchronization status.

### Inspect the backend Pod

Get its name:

```bash
kubectl get pods -n istio-demo -l app=backend
```

Set the Pod name:

```bash
BACKEND_POD=$(kubectl get pod -n istio-demo \
  -l app=backend \
  -o jsonpath='{.items[0].metadata.name}')
```

Describe the workload:

```bash
istioctl x describe pod "$BACKEND_POD" -n istio-demo
```

### Inspect Envoy listeners

```bash
istioctl proxy-config listeners "$BACKEND_POD" -n istio-demo
```

Listeners represent the network addresses and ports on which Envoy accepts traffic.

### Inspect Envoy clusters

```bash
istioctl proxy-config clusters "$BACKEND_POD" -n istio-demo
```

Clusters represent upstream destinations that Envoy can route traffic to.

### Inspect Envoy endpoints

```bash
istioctl proxy-config endpoints "$BACKEND_POD" -n istio-demo
```

Endpoints provide the upstream workload addresses Envoy can use to reach destinations.

These commands are useful when troubleshooting service discovery and traffic-routing problems.

## 8. Explore Istio Traffic Management

In this lab, we have not yet created any `VirtualService` or `DestinationRule` resources. The application is using basic Kubernetes Service discovery with Istio's injected proxies.

In later modules, we will configure Istio resources to control how requests reach services.

| Resource | Purpose |
|---|---|
| `VirtualService` | Defines routing rules for traffic |
| `DestinationRule` | Defines policies for traffic sent to a destination, including subsets and load-balancing settings |
| `Gateway` | Configures traffic entering or leaving the mesh at an Istio gateway |
| `PeerAuthentication` | Configures workload-to-workload mTLS requirements |
| `AuthorizationPolicy` | Defines access-control rules for workloads |

Check whether any Istio routing resources exist in the lab namespace:

```bash
kubectl get virtualservices,destinationrules -n istio-demo
```

If none exist, that is expected for this basic lab.

## 9. Troubleshooting

### Issue 1: The Pods are stuck in Pending

Check the Pod details:

```bash
kubectl describe pod <pod-name> -n istio-demo
```

Look for scheduling problems, insufficient cluster resources, or node constraints.

### Issue 2: The Pods are stuck in ImagePullBackOff

Check the events:

```bash
kubectl describe pod <pod-name> -n istio-demo
```

Check whether the container image can be pulled and whether your cluster has network access to the image registry.

### Issue 3: The `istio-proxy` container is missing

Check the namespace label:

```bash
kubectl get namespace istio-demo --show-labels
```

Confirm that the injection webhook is healthy:

```bash
kubectl get pods -n istio-system
```

If necessary, recreate the application Pods after resolving the issue.

### Issue 4: The frontend cannot reach the backend

Verify that the backend Pod is ready:

```bash
kubectl get pods -n istio-demo -l app=backend
```

Check the Service:

```bash
kubectl get service backend -n istio-demo
```

Check the Service endpoints:

```bash
kubectl get endpoints backend -n istio-demo
```

If your cluster supports EndpointSlices, you can inspect them too:

```bash
kubectl get endpointslices -n istio-demo
```

Check the backend logs:

```bash
kubectl logs deployment/backend -n istio-demo -c http-echo
```

Inspect proxy status:

```bash
istioctl proxy-status
```

Run Istio configuration analysis:

```bash
istioctl analyze -n istio-demo
```

### Issue 5: The request times out or returns an error

Inspect the frontend and backend Pods, verify that both sidecars are running, and review the Envoy configuration. Also check whether network policies or other cluster-level restrictions are blocking the traffic.

```bash
kubectl get pods -n istio-demo
kubectl logs "$FRONTEND_POD" -n istio-demo -c istio-proxy
kubectl logs "$BACKEND_POD" -n istio-demo -c istio-proxy
```

## 10. Clean Up

Delete the lab namespace and its resources:

```bash
kubectl delete namespace istio-demo
```

This removes the frontend, backend, Service, and other resources created in the namespace.

Istio itself remains installed in the cluster.

## Key Takeaways

- Istio can inject Envoy sidecars into eligible Pods through namespace-level configuration.
- Kubernetes Services and DNS continue to provide service discovery.
- Envoy proxies participate in application traffic handling.
- `istioctl proxy-status` and `istioctl proxy-config` help inspect the data plane.
- Istio traffic policies can be introduced without replacing Kubernetes Deployments and Services.
- Basic connectivity is only the first step; later modules cover traffic splitting, security, resilience, and observability.

## Interview Questions

1. How does automatic sidecar injection work in Istio?
2. Why must a Pod be recreated after enabling namespace injection?
3. What is the role of Envoy in an Istio service mesh?
4. Does Istio replace Kubernetes Services?
5. How does a frontend Pod discover a backend Service?
6. What does `istioctl proxy-status` do?
7. What is the difference between `VirtualService` and `DestinationRule`?
8. How would you troubleshoot a frontend-to-backend timeout?
9. How can you inspect Envoy's configured listeners and clusters?
10. Does deploying Istio automatically enable every traffic-management feature?

## Next Module

Continue to **[Module 05: Traffic Management](../05-traffic-management/README.md)** to learn about routing rules, `VirtualService`, `DestinationRule`, and advanced traffic control.

## Official References

- [Istio Getting Started](https://istio.io/latest/docs/setup/getting-started/)
- [Istio Sidecar Injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [istioctl Command Reference](https://istio.io/latest/docs/reference/commands/istioctl/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)

---

**Repository:** [Istio-Zero-to-Hero](https://github.com/bhuvan-raj/Istio-Zero-to-Hero)

**Module:** 04 — Your First Istio Application
