# 11. Istio Gateways and External Traffic

## 📌 Overview

Istio gateways control traffic entering and leaving a service mesh. They provide controlled entry points for external clients and controlled exit points for workloads accessing external services.

In a Kubernetes environment, applications often need to communicate with:

- External users accessing web applications.
- Public APIs such as payment or notification services.
- Services running outside the Kubernetes cluster.
- Other services that must be accessed through a dedicated network boundary.

Istio provides **Ingress Gateways**, **Egress Gateways**, and traffic-management resources to control these communication paths.

This module explains how to expose an application through an Istio ingress gateway, register external services with `ServiceEntry`, and route outbound traffic through an egress gateway.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- What ingress and egress gateways are.
- How external traffic enters the mesh.
- The roles of `Gateway`, `VirtualService`, and `DestinationRule`.
- How to expose an application using an Istio ingress gateway.
- How to configure HTTP routing and TLS.
- How to access external services using `ServiceEntry`.
- How egress gateways control outbound traffic.
- The difference between Istio's networking API and Kubernetes Gateway API.
- How to troubleshoot gateway connectivity.

## 📚 Table of Contents

1. [Understanding Istio Gateways](#1-understanding-istio-gateways)
2. [Ingress Gateway Architecture](#2-ingress-gateway-architecture)
3. [Important Istio Resources](#3-important-istio-resources)
4. [Hands-On Lab: Expose an Application](#4-hands-on-lab-expose-an-application)
5. [Routing Traffic to Multiple Services](#5-routing-traffic-to-multiple-services)
6. [HTTPS and TLS Configuration](#6-https-and-tls-configuration)
7. [Accessing External Services](#7-accessing-external-services)
8. [Egress Gateways](#8-egress-gateways)
9. [Ingress Gateway vs Egress Gateway](#9-ingress-gateway-vs-egress-gateway)
10. [Kubernetes Gateway API](#10-kubernetes-gateway-api)
11. [Troubleshooting](#11-troubleshooting)
12. [Security Best Practices](#12-security-best-practices)
13. [Interview Questions](#13-interview-questions)
14. [Next Module](#14-next-module)

---

## 1. Understanding Istio Gateways

An Istio gateway is a data-plane proxy that manages traffic entering or leaving a service mesh.

Unlike an application sidecar, a gateway proxy is deployed separately from the application workloads it serves.

There are two main gateway use cases.

### 1.1 Ingress Gateway

An ingress gateway manages incoming traffic from outside the mesh.

Example:

A user accesses an online shopping application using a browser. The ingress gateway receives the request and routes it to the appropriate Kubernetes service.

Common use cases:

- Exposing web applications.
- Routing requests based on hostnames and paths.
- Terminating HTTPS connections.
- Applying traffic-management rules at the mesh boundary.

### 1.2 Egress Gateway

An egress gateway manages outbound traffic from the mesh to external services.

Example:

A payment service needs to communicate with an external payment API. The organization wants this traffic to pass through a dedicated gateway for monitoring and policy enforcement.

Common use cases:

- Centralizing outbound traffic.
- Monitoring external API calls.
- Applying controlled routing for external destinations.
- Providing a dedicated network exit point.

An egress gateway alone does not guarantee that all outbound traffic must pass through it. Network controls outside Istio may be needed to prevent workloads from bypassing the gateway.

---

## 2. Ingress Gateway Architecture

Consider an application deployed on Kubernetes.

Without an ingress gateway, an external client may need to access a service through a different exposure mechanism, such as a LoadBalancer service or NodePort.

With an Istio ingress gateway, traffic follows a controlled routing path.

```text
               External Client
                      |
                      | HTTP / HTTPS
                      v
             +-------------------+
             | Load Balancer     |
             +-------------------+
                      |
                      v
             +-------------------+
             | Istio Ingress     |
             | Gateway (Envoy)   |
             +-------------------+
                      |
                      | VirtualService
                      v
             +-------------------+
             | Kubernetes Service|
             +-------------------+
                      |
                      v
             +-------------------+
             | Application Pods  |
             +-------------------+
```

### How it works

1. An external client sends a request to the application's public endpoint.
2. The cloud or network load balancer forwards the request to the ingress gateway service.
3. The gateway proxy receives the request.
4. Gateway configuration determines which hosts and ports the proxy accepts.
5. A `VirtualService` determines how matching requests are routed.
6. Traffic reaches the destination service and its application pods.

The exact network path depends on the Kubernetes service type, cloud provider, and cluster networking.

---

## 3. Important Istio Resources

### 3.1 Gateway

The Istio `Gateway` resource configures the gateway proxy's exposed ports, protocols, hostnames, and TLS settings.

It does not normally define the complete application routing logic by itself.

### 3.2 VirtualService

A `VirtualService` defines routing rules such as:

- Hostname matching.
- URL path matching.
- Header matching.
- Weighted traffic splitting.
- Timeouts and retries.

It can be associated with a gateway or used for routing within the mesh.

### 3.3 DestinationRule

A `DestinationRule` defines policies applied after routing to a destination, such as:

- Destination subsets.
- Connection pools.
- Load-balancing policies.
- Outlier detection.
- TLS settings.

### 3.4 ServiceEntry

A `ServiceEntry` adds a service to Istio's service registry when it is not automatically discovered, including external APIs and services running outside the Kubernetes cluster.

### Example request flow

```text
External Request
       |
       v
    Gateway
       |
       v
 VirtualService
       |
       v
 Destination Service
       |
       v
 DestinationRule
 (when applicable)
       |
       v
 Application Workload
```

This is a conceptual overview. A `DestinationRule` is optional, and its policies apply according to the configured destination and traffic path.

---

## 4. Hands-On Lab: Expose an Application

### Objective

Deploy an application and expose it through the Istio ingress gateway.

### Prerequisites

- A Kubernetes cluster.
- Istio installed.
- `kubectl` and `istioctl`.
- An ingress gateway deployment.
- A working `LoadBalancer`, NodePort, or port-forwarding setup.

### Step 1: Verify the ingress gateway

Check the gateway pods:

```bash
kubectl get pods -n istio-system
```

Check the services:

```bash
kubectl get svc -n istio-system
```

Look for a service such as `istio-ingressgateway`.

If the ingress gateway is missing, install or enable it using the procedure appropriate for your Istio installation.

### Step 2: Create a namespace

```bash
kubectl create namespace istio-gateway
```

Enable sidecar injection for this lab:

```bash
kubectl label namespace istio-gateway istio-injection=enabled
```

This lab uses the traditional sidecar data plane for simplicity.

### Step 3: Deploy the application

Create `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: istio-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
        - name: web
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Hello from the Istio Ingress Gateway!"
          ports:
            - name: http
              containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: istio-gateway
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 5678
```

Apply the manifest:

```bash
kubectl apply -f application.yaml
```

Wait for the deployment:

```bash
kubectl rollout status deployment/web -n istio-gateway
```

Verify the service:

```bash
kubectl get svc web -n istio-gateway
kubectl get endpointslices -n istio-gateway
```

### Step 4: Create an Istio Gateway

Create `gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "web.example.com"
```

Apply it:

```bash
kubectl apply -f gateway.yaml
```

The selector assumes that the ingress gateway pods have the label `istio: ingressgateway`. Verify the actual labels in your cluster.

### Step 5: Create a VirtualService

Create `virtualservice.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: web-route
  namespace: istio-gateway
spec:
  hosts:
    - "web.example.com"
  gateways:
    - web-gateway
  http:
    - route:
        - destination:
            host: web.istio-gateway.svc.cluster.local
            port:
              number: 80
```

Apply it:

```bash
kubectl apply -f virtualservice.yaml
```

The `Gateway` configures the entry point. The `VirtualService` defines where matching requests should go.

### Step 6: Find the ingress gateway address

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

For a service with an external IP, use that address. If the external IP is pending or unavailable in your local cluster, use the access method supported by your environment.

For example, you can port-forward the gateway:

```bash
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

In another terminal, test the request:

```bash
curl -H "Host: web.example.com" http://127.0.0.1:8080/
```

Expected response:

```text
Hello from the Istio Ingress Gateway!
```

If port-forwarding fails because the service does not expose port 80 under that name, inspect the service's actual ports and adjust the command.

### Step 7: Verify routing

Check the resources:

```bash
kubectl get gateway,virtualservice -n istio-gateway
```

Inspect the gateway logs:

```bash
kubectl logs -n istio-system deployment/istio-ingressgateway
```

The deployment name can differ depending on how the gateway was installed.

### Lab result

You have exposed a Kubernetes application using an Istio ingress gateway and a `VirtualService`.

---

## 5. Routing Traffic to Multiple Services

An ingress gateway can route traffic based on hostnames, paths, and headers.

For example:

- `/products` routes to a product service.
- `/orders` routes to an order service.

Assume both services already exist in the `istio-gateway` namespace.

Create `path-routing.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: application-routes
  namespace: istio-gateway
spec:
  hosts:
    - "apps.example.com"
  gateways:
    - web-gateway
  http:
    - match:
        - uri:
            prefix: /products
      route:
        - destination:
            host: product-service.istio-gateway.svc.cluster.local
            port:
              number: 80
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service.istio-gateway.svc.cluster.local
            port:
              number: 80
```

Apply the configuration only after the corresponding services exist:

```bash
kubectl apply -f path-routing.yaml
```

### Important

The `Gateway` must accept the requested hostname, and the `VirtualService` must be attached to the correct gateway. The destination service names and ports must also be valid.

For production APIs, review path matching, authentication, authorization, and rate-limiting requirements before exposing services externally.

---

## 6. HTTPS and TLS Configuration

HTTPS protects communication between clients and the gateway.

Istio can terminate TLS at the ingress gateway, allowing the gateway to decrypt incoming HTTPS traffic and route requests to backend services.

### TLS termination flow

```text
External Client
      |
      | HTTPS
      v
Istio Ingress Gateway
      |
      | TLS termination
      |
      | HTTP or HTTPS
      v
Backend Service
```

The connection between the gateway and backend must be configured according to the application's security requirements. TLS termination at the gateway does not automatically encrypt the next network hop.

### Step 1: Create a TLS secret

For a lab, suppose you already have a certificate and private key in `tls.crt` and `tls.key`.

Create the secret in the same namespace as the gateway workload that will use it. For the default Istio ingress gateway, that is commonly `istio-system`.

```bash
kubectl create secret tls web-tls-credential \
  --cert=tls.crt \
  --key=tls.key \
  -n istio-system
```

For a production setup, use a trusted certificate and secure secret-management procedures. Do not commit private keys to Git.

### Step 2: Configure the Gateway for HTTPS

Create `https-gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: web-https-gateway
  namespace: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "web.example.com"
      tls:
        mode: SIMPLE
        credentialName: web-tls-credential
```

Apply it:

```bash
kubectl apply -f https-gateway.yaml
```

The secret must be available to the selected gateway proxy according to Istio's credential lookup rules. Verify the secret namespace, certificate hostname, and gateway configuration if TLS does not work.

### Step 3: Test HTTPS

Once the gateway is reachable on port 443, test it using the hostname that matches the certificate:

```bash
curl --resolve web.example.com:443:<GATEWAY_IP> \
  https://web.example.com/
```

Replace `<GATEWAY_IP>` with the reachable gateway IP.

For a local port-forward, use the hostname and port appropriate for your setup. Avoid disabling certificate verification in production tests.

---

## 7. Accessing External Services

Kubernetes applications frequently consume APIs outside the cluster.

Examples include:

- Payment gateways.
- Email delivery APIs.
- Cloud APIs.
- External identity providers.
- Third-party services.

Istio can register an external service using a `ServiceEntry`, allowing the mesh to apply supported traffic-management and security features to that destination.

### What is ServiceEntry?

A `ServiceEntry` adds an external service or otherwise undiscovered service to Istio's service registry.

It can describe:

- Hostnames.
- Ports and protocols.
- Whether the service is external or internal.
- How its endpoints are resolved.

### Example: Register an external HTTPS API

Create `external-service.yaml`:

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: example-api
  namespace: istio-gateway
spec:
  hosts:
    - httpbin.org
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: tls
      protocol: TLS
  resolution: DNS
```

Apply it:

```bash
kubectl apply -f external-service.yaml
```

This registers the external host with Istio. It does not, by itself, guarantee that all egress traffic is forced through a dedicated gateway.

### Test the external connection

If you have a curl-capable application pod in the mesh, run:

```bash
kubectl exec -n istio-gateway deployment/web -c web -- \
  curl -I https://httpbin.org
```

The example application container may not include `curl`. If it does not, use a dedicated curl test pod enrolled in the mesh.

For example:

```bash
kubectl run curl-client \
  -n istio-gateway \
  --image=curlimages/curl:8.10.1 \
  --restart=Never \
  --command -- sleep 3600
```

Wait for the pod to start, then test:

```bash
kubectl exec -n istio-gateway curl-client -- \
  curl -I https://httpbin.org
```

Delete the test pod when finished:

```bash
kubectl delete pod curl-client -n istio-gateway
```

Whether this request is controlled by Istio depends on the workload's enrollment and the mesh's outbound traffic configuration. By default, Istio can allow traffic to unknown external destinations using passthrough behavior, depending on the configured outbound traffic policy.

---

## 8. Egress Gateways

An egress gateway is a dedicated proxy that provides a controlled exit point for traffic leaving the mesh.

### Why use an egress gateway?

Organizations may want external traffic to pass through a dedicated network boundary to support:

- Centralized monitoring.
- Auditing.
- Egress policy enforcement.
- Consistent outbound routing.
- Integration with firewalls and network security controls.

### Egress architecture

```text
Application Pod
       |
       v
Istio Data Plane
       |
       v
Egress Gateway
       |
       v
External API
```

An egress gateway provides a controlled routing path, but additional network-level restrictions may be necessary to prevent direct connections that bypass the gateway.

### Step 1: Check whether the egress gateway exists

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

Look for a deployment or service named `istio-egressgateway`.

The egress gateway is not installed in every Istio installation profile. Enable it using the supported installation method if required.

### Step 2: Understand the routing configuration

In traditional sidecar mode, routing traffic through an egress gateway often requires coordinating:

- `ServiceEntry` to register the external destination.
- `Gateway` to configure the egress proxy.
- `DestinationRule` when destination policies or subsets are needed.
- `VirtualService` rules to route traffic to and from the egress gateway.

The exact configuration depends on the external protocol and the Istio installation.

### Step 3: Security considerations

Do not assume that creating an egress gateway forces every workload to use it.

For stronger egress control, combine Istio routing with appropriate network controls, such as firewall rules, cloud network policies, and Kubernetes network policies where supported.

Official guide:

https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/

---

## 9. Ingress Gateway vs Egress Gateway

| Feature | Ingress Gateway | Egress Gateway |
|---|---|---|
| Traffic direction | Into the mesh | Out of the mesh |
| Main purpose | Expose applications to external clients | Control outbound access to external services |
| Common use case | Public web application | External payment API |
| TLS use | Accept and optionally terminate client TLS | Forward outbound TLS or originate TLS, depending on configuration |
| Typical controls | Host routing, path routing, TLS, authorization | Destination routing, outbound monitoring, policy enforcement |
| Network boundary | Entry point | Exit point |

Remember:

- **Ingress = traffic entering the mesh.**
- **Egress = traffic leaving the mesh.**

---

## 10. Kubernetes Gateway API

Kubernetes Gateway API is a Kubernetes-native API for configuring traffic routing.

It uses resources such as:

- `GatewayClass`
- `Gateway`
- `HTTPRoute`
- `TLSRoute`
- `GRPCRoute`

Istio supports Kubernetes Gateway API alongside its own networking APIs, including `networking.istio.io` resources such as `VirtualService` and `DestinationRule`.

### Conceptual comparison

| Istio networking API | Kubernetes Gateway API |
|---|---|
| `Gateway` | `Gateway` |
| `VirtualService` | `HTTPRoute`, `GRPCRoute`, `TLSRoute`, and other route types |
| `DestinationRule` | No single direct equivalent; capabilities depend on the feature and implementation |
| `ServiceEntry` | No direct general-purpose equivalent for registering arbitrary external services |

### Important distinction

The `Gateway` resource under `networking.istio.io` is **not the same API resource** as the Kubernetes `Gateway` under `gateway.networking.k8s.io`.

Their API groups differ:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
```

Use the appropriate API consistently in each configuration.

### When should you use Gateway API?

Gateway API is a good choice when you want Kubernetes-native routing resources and clear separation between gateway infrastructure and route configuration.

Istio's own networking API remains useful for Istio-specific features and existing installations. Choose based on your requirements and the feature support of your installed version.

Official documentation:

https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/

---

## 11. Troubleshooting

### Problem 1: The external client cannot reach the application

Check the gateway pods:

```bash
kubectl get pods -n istio-system
```

Check the gateway service:

```bash
kubectl get svc -n istio-system
```

Check routing resources:

```bash
kubectl get gateway,virtualservice -n istio-gateway
```

Verify the gateway's address, exposed port, hostname, and routing configuration.

### Problem 2: Gateway returns 404

Check:

- The HTTP `Host` header or requested hostname.
- The `Gateway` resource's `hosts` configuration.
- The `VirtualService` host and gateway attachment.
- Whether the request path matches a configured route.

Inspect the proxy configuration:

```bash
istioctl proxy-status
```

### Problem 3: Gateway returns 503

Check:

- The backend service exists.
- The service port and target port are correct.
- The service has ready endpoints.
- The application pods are healthy.
- The gateway proxy has the expected route and cluster configuration.

Useful commands:

```bash
kubectl get svc,endpointslices -n istio-gateway
kubectl get pods -n istio-gateway
```

Inspect proxy clusters:

```bash
istioctl proxy-config clusters <gateway-pod> -n istio-system
```

### Problem 4: HTTPS is not working

Check:

- The TLS secret exists in the expected namespace.
- The certificate matches the requested hostname.
- The gateway has a listener on port 443.
- The gateway references the correct credential.
- The external service or load balancer exposes HTTPS.

### Problem 5: External service requests fail

Check:

- DNS resolution from the cluster.
- Network egress and firewall restrictions.
- The `ServiceEntry` host, protocol, and port.
- TLS configuration.
- The mesh's outbound traffic policy.
- Whether a proxy or network policy is blocking the request.

### Problem 6: Egress traffic bypasses the gateway

Istio routing configuration alone is not a complete network security boundary. If workloads can access external destinations directly, additional firewall or network controls may be necessary to enforce a dedicated egress path.

---

## 12. Security Best Practices

- Expose only the services that must be externally reachable.
- Use HTTPS and valid certificates for public endpoints.
- Apply authentication and authorization where required.
- Restrict accepted hostnames and routing rules.
- Validate external service definitions.
- Monitor gateway access logs, request metrics, and failures.
- Use network-level controls when outbound traffic must not bypass an egress gateway.
- Avoid storing private keys in source control.
- Restrict access to gateway configuration and TLS secrets.
- Test routing and security policies before production rollout.
- Keep gateway proxies and Istio components updated.

---

## 13. Interview Questions

### Beginner

**1. What is an Istio Gateway?**

An Istio Gateway configures a proxy at the mesh boundary to handle incoming or outgoing traffic.

**2. What is an ingress gateway?**

An ingress gateway manages traffic entering the mesh from external clients.

**3. What is an egress gateway?**

An egress gateway provides a dedicated routing point for traffic leaving the mesh.

**4. What is the difference between Gateway and VirtualService?**

A `Gateway` configures the proxy's listeners, accepted hosts, and TLS behavior. A `VirtualService` defines routing rules for matching traffic.

**5. What is ServiceEntry?**

A `ServiceEntry` registers a service in Istio's service registry, including external APIs that are not automatically discovered.

### Intermediate

**6. How does HTTPS termination work at an ingress gateway?**

The gateway presents a TLS certificate and decrypts the incoming HTTPS connection. It then routes the request to the backend according to its configuration.

**7. Why would you use an egress gateway?**

To provide a dedicated outbound traffic path for monitoring, routing, and policy enforcement.

**8. Does a ServiceEntry automatically force traffic through an egress gateway?**

No. It registers the destination with Istio. Additional routing configuration is needed to direct traffic through an egress gateway.

**9. What is the difference between Istio Gateway and Kubernetes Gateway API Gateway?**

They are different resources from different API groups. Istio's Gateway belongs to `networking.istio.io`; Kubernetes Gateway API's Gateway belongs to `gateway.networking.k8s.io`.

**10. Why might an ingress gateway return HTTP 404?**

The requested hostname or path may not match the configured Gateway and VirtualService rules, or the VirtualService may not be attached correctly.

### Advanced

**11. How would you troubleshoot HTTP 503 from an ingress gateway?**

I would inspect the backend service and EndpointSlices, verify pod readiness, inspect the gateway's routes and clusters, and check proxy logs for upstream connection failures.

**12. How can you enforce that all outbound traffic uses an egress gateway?**

I would configure Istio routing and enforce the network boundary using appropriate firewall or network policies so workloads cannot directly bypass the gateway.

**13. Can Istio Gateway be used without VirtualService?**

A Gateway can configure listeners, but application-layer routing normally requires appropriate routing configuration, such as a VirtualService or a supported Gateway API route.

**14. What is the role of DestinationRule in gateway traffic management?**

A DestinationRule can define destination subsets, load balancing, connection pools, outlier detection, and applicable TLS policies.

**15. How would you expose multiple applications through one ingress gateway?**

I would configure host- or path-based routing rules and route each matching request to the correct Kubernetes service.

---

## 14. Next Module

Continue to:

[**12. Kubernetes Integration →**](../12-kubernetes-integration/README.md)

## 📖 Official Documentation

- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Ingress Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/)
- [Kubernetes Gateway API](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)
- [ServiceEntry Reference](https://istio.io/latest/docs/reference/config/networking/service-entry/)
- [Egress Gateways](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)
- [External Service Access](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/)

---

**Previous:** [10. Ambient Mesh](../10-ambient-mesh/README.md)  
**Next:** [12. Kubernetes Integration](../12-kubernetes-integration/README.md)
