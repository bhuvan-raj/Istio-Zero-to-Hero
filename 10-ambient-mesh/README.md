# 10. Istio Ambient Mesh

## 📌 Overview

Istio Ambient Mesh is a sidecarless service mesh architecture that provides secure communication, traffic management, and observability for Kubernetes workloads without requiring an Envoy sidecar in every application pod.

Unlike traditional Istio sidecar mode, Ambient Mesh separates service mesh functionality into two layers:

- **ztunnel:** A lightweight, node-level proxy that provides secure Layer 4 (L4) connectivity, workload identity, mutual TLS (mTLS), and Layer 4 authorization.
- **Waypoint proxy:** An optional Envoy-based proxy that provides Layer 7 (L7) capabilities such as HTTP routing, HTTP-aware authorization, retries, timeouts, and advanced telemetry.

This layered architecture allows teams to adopt mesh security first and enable additional Layer 7 features when needed.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What Istio Ambient Mesh is and why it exists.
- The difference between sidecar and ambient architectures.
- The roles of ztunnel and waypoint proxies.
- How ambient traffic routing works.
- How mTLS and workload identity work in Ambient Mesh.
- How to install Istio with ambient support.
- How to enroll Kubernetes namespaces into the mesh.
- How to deploy a waypoint proxy.
- How to apply Layer 4 and Layer 7 authorization.
- How to troubleshoot ambient workloads.

## 📚 Table of Contents

1. [What Is Ambient Mesh?](#1-what-is-ambient-mesh)
2. [Why Ambient Mesh?](#2-why-ambient-mesh)
3. [Ambient Mesh Architecture](#3-ambient-mesh-architecture)
4. [Sidecar vs Ambient Mesh](#4-sidecar-vs-ambient-mesh)
5. [Understanding ztunnel](#5-understanding-ztunnel)
6. [Understanding Waypoint Proxies](#6-understanding-waypoint-proxies)
7. [How Ambient Traffic Flows](#7-how-ambient-traffic-flows)
8. [Install Istio with Ambient Support](#8-install-istio-with-ambient-support)
9. [Hands-On Lab: Deploy an Ambient Application](#9-hands-on-lab-deploy-an-ambient-application)
10. [Verify Ambient Enrollment](#10-verify-ambient-enrollment)
11. [Deploy a Waypoint Proxy](#11-deploy-a-waypoint-proxy)
12. [Security and Authorization](#12-security-and-authorization)
13. [Observability](#13-observability)
14. [Troubleshooting](#14-troubleshooting)
15. [Best Practices](#15-best-practices)
16. [Interview Questions](#16-interview-questions)
17. [Next Module](#17-next-module)

---

## 1. What Is Ambient Mesh?

Ambient Mesh is an Istio data-plane architecture that allows Kubernetes workloads to participate in a service mesh without injecting an Envoy sidecar into each application pod.

In traditional sidecar mode, every enrolled application pod typically contains two containers:

- The application container.
- The Envoy sidecar container.

In Ambient Mesh, application pods do not need an injected Envoy sidecar. Instead, node-level ztunnel proxies handle secure Layer 4 traffic, while optional waypoint proxies provide Layer 7 functionality.

### Traditional sidecar architecture

```text
+--------------------------+
| Application Pod          |
|                          |
|  +--------------------+  |
|  | Application        |  |
|  +--------------------+  |
|            |             |
|  +--------------------+  |
|  | Envoy Sidecar      |  |
|  +--------------------+  |
+--------------------------+
             |
             v
       Other Service
```

### Ambient Mesh architecture

```text
+------------------+          +------------------+
| Application Pod  |          | Application Pod  |
|                  |          |                  |
| Application      |          | Application      |
+------------------+          +------------------+
         |                              ^
         v                              |
+------------------+          +------------------+
| ztunnel          |<-------->| ztunnel          |
| Node-level proxy |   mTLS   | Node-level proxy |
+------------------+          +------------------+
```

A waypoint can be introduced when Layer 7 processing is required.

The main idea is that **mesh functionality is separated from individual application pods**.

Official documentation: https://istio.io/latest/docs/ambient/overview/

## 2. Why Ambient Mesh?

Sidecar proxies provide powerful service mesh functionality, but operating a proxy in every application pod introduces additional lifecycle and resource considerations.

For example, a Kubernetes application with 100 pods may have 100 Envoy sidecars to manage in addition to its application containers.

Ambient Mesh offers a different operational model by using shared node-level proxies and optional waypoint proxies.

### Benefits

- No Envoy sidecar injection is required for ambient-enrolled application pods.
- Secure service-to-service communication using mTLS.
- Layer 4 authorization through ztunnel.
- Optional Layer 7 processing through waypoint proxies.
- Independent lifecycle management for ambient components.
- A gradual adoption path: secure connectivity first, advanced traffic management when needed.

### Important consideration

Ambient Mesh does not eliminate proxy infrastructure or all operational overhead. ztunnel runs on cluster nodes, and waypoint proxies consume resources when deployed. The architecture changes how proxies are deployed and managed.

---

## 3. Ambient Mesh Architecture

Ambient Mesh consists of control-plane components and data-plane components.

### Control plane

**Istiod** manages mesh configuration, service discovery, and workload certificates. It distributes the necessary configuration to the data-plane components.

### Data plane

The ambient data plane consists primarily of:

- **ztunnel:** Node-level Layer 4 proxy.
- **Waypoint proxy:** Optional Layer 7 proxy.

### Architecture diagram

```text
                     Control Plane
                   +----------------+
                   |     Istiod     |
                   +----------------+
                     |            |
              Configuration   Certificates
                     |            |
        +------------+------------+------------+
        |                         |            |
        v                         v            v
+----------------+       +----------------+  Waypoint
| ztunnel        |       | ztunnel        |  (optional)
| Node 1         |       | Node 2         |     |
+----------------+       +----------------+     |
        |                         |             |
        v                         v             v
+----------------+       +----------------+  HTTP/L7
| Application A  |       | Application B  |  policies
+----------------+       +----------------+
```

The exact request path depends on workload enrollment, waypoint configuration, and the traffic destination.

### Key components

| Component | Responsibility |
|---|---|
| Istiod | Control-plane configuration and certificate management |
| ztunnel | Secure Layer 4 connectivity and supported Layer 4 policies |
| Waypoint | Layer 7 routing, authorization, and telemetry |
| Istio CNI | Configures traffic redirection for ambient workloads |
| Application pod | Runs the application without an injected sidecar |

---

## 4. Sidecar vs Ambient Mesh

| Feature | Sidecar mode | Ambient Mesh |
|---|---|---|
| Proxy deployment | Envoy sidecar in each enrolled pod | Node-level ztunnel, optional waypoint |
| Sidecar injection | Typically required | Not required |
| mTLS | Supported | Supported |
| Layer 4 authorization | Supported | Supported through ztunnel |
| Layer 7 authorization | Supported | Requires waypoint processing |
| HTTP routing | Supported | Requires waypoint processing for ambient L7 routing |
| HTTP retries and timeouts | Supported | Requires appropriate Layer 7 processing |
| Proxy lifecycle | Sidecars managed alongside workloads | ztunnel and waypoint lifecycle managed separately |
| Resource model | Proxy resources per pod | Shared node-level proxy plus optional waypoints |

Both architectures are part of Istio. They can coexist within a mesh, subject to the feature support and configuration of the installed release.

---

## 5. Understanding ztunnel

ztunnel means **Zero Trust tunnel**. It is a lightweight, purpose-built proxy that runs as a Kubernetes DaemonSet in Ambient Mesh.

A DaemonSet normally ensures that a ztunnel pod runs on each eligible node.

### Responsibilities

- Secure transport between enrolled workloads.
- Workload identity and mTLS.
- Layer 4 authorization.
- Traffic forwarding.
- Layer 4 telemetry.

ztunnel is intentionally designed not to parse application HTTP headers or provide general Layer 7 HTTP routing.

### Inspect ztunnel

List ztunnel pods:

```bash
kubectl get pods -n istio-system -l app=ztunnel
```

Inspect the DaemonSet:

```bash
kubectl get daemonset -n istio-system
```

View recent logs:

```bash
kubectl logs -n istio-system -l app=ztunnel --tail=100
```

Inspect ambient workloads:

```bash
istioctl ztunnel-config workloads
```

Inspect available ztunnel commands for your installed version:

```bash
istioctl ztunnel-config --help
```

### Key point

ztunnel provides the secure Layer 4 foundation of Ambient Mesh. You do not need a waypoint just to enable ambient mTLS.

---

## 6. Understanding Waypoint Proxies

A waypoint is an Envoy-based proxy deployed outside application pods to provide Layer 7 processing for ambient workloads.

Waypoints can be deployed and scaled independently from the application workloads they serve.

### Responsibilities

- HTTP path and header-based routing.
- Layer 7 authorization.
- HTTP traffic splitting.
- Retries and timeouts.
- HTTP-level telemetry and access logs.
- Other supported Layer 7 features.

### When is a waypoint required?

A waypoint is generally required when you need ambient Layer 7 functionality, such as allowing `GET /orders` while denying `DELETE /orders`.

A waypoint is not required for every ambient workload.

### Waypoint architecture

```text
Frontend Pod
     |
     v
Source Node's ztunnel
     |
     | Secure HBONE tunnel
     v
+----------------------+
| Waypoint Proxy       |
|                      |
| HTTP Routing         |
| L7 Authorization     |
| Retries / Timeouts   |
+----------------------+
     |
     | Secure HBONE tunnel
     v
Destination Node's ztunnel
     |
     v
Backend Pod
```

This diagram shows a conceptual path when a waypoint is configured for the destination. The actual path depends on the workload and service configuration.

### Waypoint vs ingress gateway

| Waypoint | Ingress Gateway |
|---|---|
| Handles Layer 7 processing for ambient workloads | Handles traffic entering the mesh |
| Used for internal service-to-service policies and routing | Commonly exposes services to external clients |
| Usually associated with workloads or namespaces | Usually configured at the mesh boundary |

---

## 7. How Ambient Traffic Flows

Consider two services:

- `frontend`
- `backend`

Both services are enrolled in Ambient Mesh.

### Without a waypoint

1. The frontend sends a request to the backend.
2. Traffic is transparently redirected to the node-local ztunnel.
3. The ambient data plane identifies the destination and applies relevant Layer 4 policy.
4. When both workloads participate in the mesh, traffic can be transported using authenticated mTLS through HBONE.
5. The destination-side data plane forwards the request to the backend.

### With a waypoint

1. The frontend sends a request to the backend.
2. Traffic is intercepted by the ambient data plane.
3. The request is forwarded through the waypoint associated with the destination.
4. The waypoint applies the configured Layer 7 routing and authorization.
5. Traffic is forwarded toward the destination workload through the ambient data plane.

### What is HBONE?

HBONE stands for **HTTP-Based Overlay Network Environment**.

It is an HTTP CONNECT-based tunneling protocol used by Istio Ambient Mesh to transport traffic securely between mesh components.

HBONE is a transport mechanism for ambient traffic. It is not the same as the application's HTTP request itself.

---

## 8. Install Istio with Ambient Support

### Prerequisites

- A working Kubernetes cluster.
- `kubectl` configured for that cluster.
- `istioctl` matching the intended Istio release.
- Permissions to install Istio and its networking components.

### Step 1: Install the Istio CLI

Follow the official installation instructions for your operating system:

https://istio.io/latest/docs/ambient/install/istioctl/

Verify the installation:

```bash
istioctl version
```

### Step 2: Install the ambient profile

For a new lab cluster, install Istio using the ambient profile:

```bash
istioctl install --set profile=ambient -y
```

This installs the components required for Ambient Mesh.

Check the status:

```bash
kubectl get pods -n istio-system
```

Verify the installation:

```bash
istioctl verify-install
```

Wait for the relevant components to become ready before continuing.

**Important:** If your cluster already has Istio installed, do not blindly run a new installation command. Follow the official upgrade or migration instructions for your current installation.

---

## 9. Hands-On Lab: Deploy an Ambient Application

### Objective

Deploy a frontend and backend, enroll their namespace in Ambient Mesh, and test service-to-service communication without sidecar injection.

### Step 1: Create the namespace

```bash
kubectl create namespace istio-ambient
```

Enroll it in Ambient Mesh:

```bash
kubectl label namespace istio-ambient \
  istio.io/dataplane-mode=ambient
```

Verify the label:

```bash
kubectl get namespace istio-ambient --show-labels
```

### Step 2: Create the application

Create a file named `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: istio-ambient
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
        - name: backend
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Hello from Ambient Mesh!"
          ports:
            - name: http
              containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-ambient
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
  namespace: istio-ambient
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
            - sh
            - -c
            - sleep 3650d
```

Apply the manifest:

```bash
kubectl apply -f application.yaml
```

Wait for both deployments:

```bash
kubectl rollout status deployment/backend -n istio-ambient
kubectl rollout status deployment/frontend -n istio-ambient
```

### Step 3: Test connectivity

Run a request from the frontend:

```bash
kubectl exec -n istio-ambient deployment/frontend -c curl -- \
  curl -sS http://backend.istio-ambient.svc.cluster.local:5678
```

Expected response:

```text
Hello from Ambient Mesh!
```

### Step 4: Verify the pods

```bash
kubectl get pods -n istio-ambient
```

Confirm that the application pods do not contain an injected `istio-proxy` sidecar.

This demonstrates the sidecarless workload model.

---

## 10. Verify Ambient Enrollment

### Step 1: Inspect ztunnel workloads

```bash
istioctl ztunnel-config workloads
```

Find the `frontend` and `backend` workloads in the output.

### Step 2: Inspect ztunnel pods

```bash
kubectl get pods -n istio-system -l app=ztunnel
```

### Step 3: Verify mTLS

A successful HTTP request alone does not prove that mTLS is active.

Use the verification procedure documented for your Istio version, inspect the relevant ambient configuration, and verify that the traffic follows the expected data-plane path.

Official guide:

https://istio.io/latest/docs/ambient/usage/verify-mtls-enabled/

### Step 4: Observe traffic

Generate repeated requests:

```bash
kubectl exec -n istio-ambient deployment/frontend -c curl -- \
  sh -c 'for i in $(seq 1 20); do curl -sS http://backend.istio-ambient.svc.cluster.local:5678; echo; done'
```

Inspect ztunnel logs and available telemetry to see how the mesh handles the traffic.

---

## 11. Deploy a Waypoint Proxy

A waypoint is required when you want ambient Layer 7 processing.

### Step 1: Create the waypoint

```bash
istioctl waypoint apply \
  --namespace istio-ambient \
  --name waypoint
```

Check the Gateway resource:

```bash
kubectl get gateway -n istio-ambient
```

Check the waypoint pods:

```bash
kubectl get pods -n istio-ambient
```

The waypoint is managed using Kubernetes Gateway API resources. If the Gateway API CRDs are missing, install the supported CRDs following the official Istio instructions.

### Step 2: Associate the backend with the waypoint

For this lab, associate the namespace with the waypoint:

```bash
kubectl label namespace istio-ambient \
  istio.io/use-waypoint=waypoint
```

This setting applies to eligible workloads in the namespace. Use workload-level enrollment when you need more selective behavior.

### Step 3: Test again

```bash
kubectl exec -n istio-ambient deployment/frontend -c curl -- \
  curl -sS http://backend.istio-ambient.svc.cluster.local:5678
```

Inspect the waypoint logs and verify that the backend traffic follows the intended path.

```bash
kubectl get pods -n istio-ambient
kubectl logs -n istio-ambient deployment/waypoint --tail=100
```

If the generated workload name differs, use the actual waypoint pod or deployment name.

**Important:** Creating a waypoint does not automatically guarantee that every request uses it. Confirm that the target workload is associated with the waypoint and that the traffic is routed through it.

---

## 12. Security and Authorization

Ambient Mesh supports zero-trust security through workload identity, mTLS, and authorization policies.

### Layer 4 authorization

ztunnel supports Layer 4 authorization. Policies can restrict connectivity based on attributes such as workload identity and destination port.

### Layer 7 authorization

Waypoint proxies support Layer 7 authorization, including policies based on HTTP methods and paths.

For example:

- Allow `GET /orders`.
- Deny `DELETE /orders`.
- Allow requests only from a specific workload identity.

### Important considerations

- Verify that the workload is enrolled in Ambient Mesh.
- Use ztunnel-supported policies for Layer 4 controls.
- Deploy a waypoint for policies that require HTTP attributes.
- Test both allowed and denied requests.
- Avoid applying restrictive policies to shared namespaces without validating their impact.

Official documentation:

- [Ambient Mesh security](https://istio.io/latest/docs/ambient/)
- [Layer 4 security policy](https://istio.io/latest/docs/ambient/usage/l4-policy/)

---

## 13. Observability

Ambient Mesh supports observability through the ambient data-plane components.

### What can you monitor?

- Request rates and failures.
- ztunnel health.
- Waypoint health.
- Workload communication.
- Authorization behavior.
- Layer 4 and Layer 7 telemetry, depending on the processing path.

### Inspect ztunnel

```bash
kubectl get pods -n istio-system -l app=ztunnel
```

```bash
kubectl logs -n istio-system -l app=ztunnel --tail=100
```

### Inspect waypoint

```bash
kubectl get pods -n istio-ambient
```

```bash
kubectl logs -n istio-ambient deployment/waypoint --tail=100
```

Prometheus, Grafana, and Kiali can be used when installed and correctly configured. Do not assume that every observability addon is included in the core ambient installation.

For more details, see:

https://istio.io/latest/docs/ambient/architecture/data-plane/

---

## 14. Troubleshooting

### Problem 1: The frontend cannot reach the backend

Check the workloads and services:

```bash
kubectl get pods,svc -n istio-ambient
kubectl get endpointslices -n istio-ambient
```

Check the namespace labels:

```bash
kubectl get namespace istio-ambient --show-labels
```

Check the ambient data plane:

```bash
kubectl get pods -n istio-system -l app=ztunnel
istioctl ztunnel-config workloads
```

### Problem 2: The workload is missing from ztunnel configuration

Verify that:

- The namespace has `istio.io/dataplane-mode=ambient`.
- ztunnel is healthy on the node.
- The Istio CNI component is functioning.
- The workload is not excluded from ambient traffic redirection.
- The workload has been reconciled by the data plane.

### Problem 3: Layer 7 policies are not working

Verify that:

- The waypoint exists and is ready.
- The workload is associated with the intended waypoint.
- Traffic actually passes through the waypoint.
- The policy selects the correct namespace and workload.
- The policy uses features supported by the installed Istio version.

### Problem 4: mTLS cannot be verified

Check:

- Whether the workloads are enrolled in the mesh.
- Whether ztunnel is healthy.
- Whether traffic follows the ambient data plane.
- Whether the expected workload identity and certificates are available.
- The official mTLS verification output for the installed version.

### Problem 5: The waypoint is not ready

Inspect the Gateway resource:

```bash
kubectl describe gateway waypoint -n istio-ambient
```

Inspect the pods and events:

```bash
kubectl get pods -n istio-ambient
kubectl get events -n istio-ambient --sort-by=.lastTimestamp
```

Check the Gateway API CRDs, image availability, resource limits, and Istio control-plane logs.

---

## 15. Best Practices

- Start with ambient L4 security before enabling optional L7 functionality.
- Deploy waypoints only where Layer 7 processing is required.
- Verify mTLS rather than assuming it is enabled.
- Validate authorization policies with both allowed and denied requests.
- Monitor ztunnel, waypoints, and Istiod.
- Plan capacity for node-level proxies and waypoint workloads.
- Follow official upgrade and migration guidance for existing installations.
- Test application behavior before and after changing data-plane modes.
- Understand the feature support and limitations of the specific Istio release.

---

## 16. Interview Questions

### Beginner

**1. What is Istio Ambient Mesh?**

Ambient Mesh is a sidecarless Istio architecture that provides mesh capabilities using node-level ztunnel proxies and optional waypoint proxies.

**2. What is ztunnel?**

ztunnel is a node-level proxy that provides secure Layer 4 connectivity, workload identity, mTLS, and supported Layer 4 authorization.

**3. What is a waypoint proxy?**

A waypoint is an Envoy-based proxy that provides Layer 7 processing, including HTTP routing and HTTP-aware authorization.

**4. Does Ambient Mesh require sidecar injection?**

No. Ambient-enrolled application pods do not require injected Envoy sidecars.

**5. Is a waypoint mandatory?**

No. A waypoint is optional and is deployed when Layer 7 capabilities are required.

### Intermediate

**6. What is the difference between ztunnel and a waypoint?**

ztunnel handles Layer 4 security and connectivity, while a waypoint handles Layer 7 traffic processing and policies.

**7. How do you enroll a namespace in Ambient Mesh?**

Apply the label `istio.io/dataplane-mode=ambient` to the namespace.

**8. What is HBONE?**

HBONE is Istio's HTTP CONNECT-based tunneling protocol used to transport traffic between ambient mesh components.

**9. How do you verify ambient enrollment?**

Use `istioctl ztunnel-config workloads` and inspect the namespace labels and data-plane health.

**10. Can Ambient Mesh provide mTLS without a waypoint?**

Yes. ztunnel provides the secure Layer 4 foundation without requiring a waypoint.

### Advanced

**11. Why might an HTTP authorization policy fail in Ambient Mesh?**

The workload may not be associated with a waypoint, traffic may bypass Layer 7 processing, or the policy may target the wrong workload.

**12. Can sidecar and ambient workloads coexist?**

Yes. Istio supports mixed data-plane modes, subject to the configuration and supported features of the installed release.

**13. Why does Ambient Mesh use ztunnel instead of Envoy sidecars for every pod?**

It separates secure Layer 4 connectivity from optional Layer 7 processing, changing the proxy lifecycle and reducing the need to manage a sidecar in every application pod.

**14. How would you troubleshoot a workload missing from Ambient Mesh?**

Check namespace enrollment, ztunnel health, CNI configuration, traffic redirection, and `istioctl ztunnel-config workloads`.

**15. When would you choose Ambient Mesh over sidecar mode?**

I would consider Ambient Mesh when its sidecarless deployment model fits the workload, while validating required traffic-management features, security policies, observability, and operational constraints.

---

## 17. Next Module

Continue to:

[**11. Gateways and External Traffic →**](../11-gateways-external-traffic/README.md)

## 📖 Official Documentation

- [Ambient Mesh Overview](https://istio.io/latest/docs/ambient/overview/)
- [Ambient Mesh Architecture](https://istio.io/latest/docs/ambient/architecture/)
- [Ambient Data Plane](https://istio.io/latest/docs/ambient/architecture/data-plane/)
- [Install Istio with Ambient Support](https://istio.io/latest/docs/ambient/install/istioctl/)
- [Waypoint Proxies](https://istio.io/latest/docs/ambient/usage/waypoint/)
- [Layer 4 Security Policy](https://istio.io/latest/docs/ambient/usage/l4-policy/)
- [Ambient Troubleshooting](https://istio.io/latest/docs/ambient/)

---

**Previous:** [09. Observability](../09-observability/README.md)  
**Next:** [11. Gateways and External Traffic](../11-gateways-external-traffic/README.md)
