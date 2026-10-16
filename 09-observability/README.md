# 09. Istio Observability

## 📌 Overview

Observability is the ability to understand what is happening inside an application by analyzing its **metrics, logs, and traces**.

In a microservices architecture, a single request may pass through multiple services. When a request becomes slow or fails, identifying the root cause can be difficult.

Istio improves observability by collecting telemetry from service-to-service communication through its Envoy proxies, often without requiring application code changes.

This module covers how to monitor, troubleshoot, and visualize traffic in an Istio service mesh.

## 🎯 Learning Objectives

By the end of this module, you will understand:

- The three pillars of observability.
- How Istio collects service metrics.
- How to monitor traffic using Prometheus and Grafana.
- How to enable and inspect Envoy access logs.
- How distributed tracing works.
- How to visualize service dependencies using Kiali.
- How to troubleshoot errors and latency in a service mesh.
- How to use the Istio Telemetry API.

## 📚 Table of Contents

1. [What Is Observability?](#1-what-is-observability)
2. [Istio Observability Architecture](#2-istio-observability-architecture)
3. [The Three Pillars](#3-the-three-pillars-of-observability)
4. [Istio Metrics](#4-istio-metrics)
5. [Prometheus and Grafana](#5-prometheus-and-grafana)
6. [Access Logs](#6-access-logs)
7. [Distributed Tracing](#7-distributed-tracing)
8. [Visualizing the Mesh with Kiali](#8-visualizing-the-mesh-with-kiali)
9. [Istio Telemetry API](#9-istio-telemetry-api)
10. [Hands-On Lab](#10-hands-on-lab)
11. [Troubleshooting](#11-troubleshooting)
12. [Best Practices](#12-best-practices)
13. [Interview Questions](#13-interview-questions)
14. [Next Module](#14-next-module)

---

## 1. What Is Observability?

Observability is the ability to understand a system's internal state using the information it exposes.

For example, suppose an application running on Kubernetes consists of these services:

- Frontend
- Order service
- Payment service
- Inventory service

A user places an order, but the request takes five seconds to complete.

Observability helps answer questions such as:

- Which service is causing the delay?
- How many requests are failing?
- What is the response time of each service?
- Which service is communicating with another service?
- Are requests failing because of application errors or network problems?

Istio generates telemetry from traffic passing through its proxies, helping platform engineers investigate these questions.

## 2. Istio Observability Architecture

A typical observability architecture looks like this:

```text
                 Application Traffic
                         |
                         v
                 +---------------+
                 | Frontend Pod  |
                 | Application   |
                 | Envoy Proxy   |
                 +---------------+
                         |
                         v
                 +---------------+
                 | Backend Pod   |
                 | Application   |
                 | Envoy Proxy   |
                 +---------------+
                         |
                         v
                  Istio Telemetry
                         |
           +-------------+-------------+
           |             |             |
           v             v             v
        Metrics         Logs         Traces
           |             |             |
           v             v             v
      Prometheus      Loki/Log       Jaeger/
           |          Collector      Tempo
           v             |             |
        Grafana <---------+-------------+
```

The diagram represents a possible architecture, not a default installation. Each telemetry backend must be installed and configured as required.

### Important components

| Component | Purpose |
|---|---|
| Envoy | Generates telemetry about proxied traffic |
| Istiod | Manages mesh configuration and control-plane telemetry |
| Prometheus | Collects and stores time-series metrics |
| Grafana | Visualizes metrics through dashboards |
| Loki | Stores and queries logs |
| Jaeger | Collects and visualizes distributed traces |
| Tempo | Stores distributed traces |
| OpenTelemetry Collector | Receives, processes, and exports telemetry |
| Kiali | Visualizes service relationships and mesh health |

Istio supports multiple telemetry integrations. You do not need to install every tool in the table.

## 3. The Three Pillars of Observability

### 3.1 Metrics

Metrics are numerical measurements collected over time.

Examples:

- Requests per second
- HTTP error rate
- Request duration
- CPU and memory usage
- Number of active connections

Metrics help you understand the overall health and performance of the application.

**Example:** If the payment service's HTTP 5xx error rate increases, an alert can notify the DevOps team.

### 3.2 Logs

Logs are records of individual events.

Example:

```text
GET /orders 200
GET /payments 500
POST /inventory 503
```

Logs help identify what happened during a particular request or operation.

In Istio, Envoy access logs can contain information such as HTTP methods, response codes, request duration, upstream details, and response flags.

### 3.3 Distributed Traces

A distributed trace follows a request as it travels through multiple services.

Example:

```text
Request: POST /checkout
|
+-- Frontend:       20 ms
|
+-- Order Service:  50 ms
|
+-- Payment Service: 900 ms
|
+-- Inventory:      30 ms
```

In this example, the payment service contributes most of the latency.

Distributed tracing helps identify slow dependencies and understand the flow of a request across service boundaries.

### Comparison

| Pillar | Answers | Common tools |
|---|---|---|
| Metrics | How much? How often? How fast? | Prometheus, Grafana |
| Logs | What happened? | Loki, Elasticsearch |
| Traces | Where did the request spend time? | Jaeger, Tempo |
| Mesh visualization | How are services connected? | Kiali |

---

## 4. Istio Metrics

Istio collects metrics about requests passing through its proxies.

Common metric categories include:

- Request count
- Request duration
- Request size
- Response size
- TCP traffic
- Control-plane health

The exact metrics exposed depend on the Istio version, telemetry configuration, protocol, and workload.

### Important metric names

| Metric | Description |
|---|---|
| `istio_requests_total` | Total number of requests |
| `istio_request_duration_milliseconds` | Request duration distribution in versions/configurations that expose this metric |
| `istio_request_bytes` | Request size distribution |
| `istio_response_bytes` | Response size distribution |
| `istio_tcp_sent_bytes_total` | Total bytes sent for TCP traffic |
| `istio_tcp_received_bytes_total` | Total bytes received for TCP traffic |

Check your actual Prometheus targets and metric names before using these in dashboards. Metric names and labels can vary across releases and configurations.

### Useful labels

Depending on the metric, Istio may expose labels such as:

- `source_workload`
- `destination_workload`
- `source_namespace`
- `destination_namespace`
- `response_code`
- `reporter`

These labels let you filter metrics by workload, namespace, response code, and traffic direction.

### Example PromQL queries

**Total request rate over the last five minutes:**

```promql
sum(rate(istio_requests_total[5m]))
```

**Rate of HTTP 5xx responses:**

```promql
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
```

**Request rate grouped by destination workload:**

```promql
sum by (destination_workload) (
  rate(istio_requests_total{reporter="destination"}[5m])
)
```

**Percentage of HTTP 5xx responses:**

```promql
100 *
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
/
clamp_min(sum(rate(istio_requests_total[5m])), 1e-9)
```

These queries assume the relevant metric and labels are present. The error-rate query is meaningful for HTTP request metrics and may need adjustment for your environment.

---

## 5. Prometheus and Grafana

### 5.1 What Is Prometheus?

Prometheus is an open-source monitoring system that collects and stores metrics as time-series data.

It can scrape metrics endpoints exposed by applications, Envoy proxies, and Istio components.

Prometheus supports PromQL, a query language for analyzing time-series data.

### 5.2 What Is Grafana?

Grafana is a visualization and dashboard platform.

It connects to data sources such as Prometheus and displays information through charts, graphs, tables, and alerts.

### 5.3 How they work together

```text
Istio / Envoy Metrics
         |
         v
    Prometheus
         |
      PromQL
         |
         v
      Grafana
         |
         v
 Dashboards and Alerts
```

### 5.4 Install optional Istio addons

The Istio distribution includes sample addon manifests in its `samples/addons` directory. If you have downloaded the Istio distribution, navigate to that directory.

```bash
cd istio-<version>/samples/addons
```

Check which addon manifests are available:

```bash
ls
```

If the Prometheus and Grafana manifests are present, you can install them into `istio-system`:

```bash
kubectl apply -f prometheus.yaml
kubectl apply -f grafana.yaml
```

These sample addons are intended for learning and demonstration. They are not a production-ready monitoring stack.

Check the deployment status:

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

If you already have a monitoring stack, use its existing configuration instead of installing duplicate components.

### 5.5 Access the Prometheus dashboard

If a Prometheus service exists in `istio-system`, inspect its name:

```bash
kubectl get svc -n istio-system
```

Use the actual service name in the port-forward command. For example:

```bash
kubectl port-forward -n istio-system svc/prometheus 9090:9090
```

Open the Prometheus UI:

```text
http://localhost:9090
```

Try querying:

```promql
istio_requests_total
```

If no results appear, generate traffic and check whether Prometheus is scraping the appropriate targets.

### 5.6 Access the Grafana dashboard

If the Grafana service is available:

```bash
kubectl port-forward -n istio-system svc/grafana 3000:3000
```

Open:

```text
http://localhost:3000
```

The exact service name, login settings, dashboards, and data-source configuration depend on the installation. Configure Prometheus as a Grafana data source if it is not already configured.

### Important

Installing the addon manifests does not guarantee that every dashboard or data source is configured exactly as you need. Verify the service names, endpoints, and dashboard setup in your environment.

---

## 6. Access Logs

Access logs provide details about requests handled by Envoy proxies.

They are useful when investigating:

- HTTP 4xx and 5xx responses
- Connection failures
- Unexpected routing behavior
- Request latency
- Upstream service problems

### 6.1 Enable access logging with the Telemetry API

Istio's `Telemetry` resource can enable access logging at mesh, namespace, or workload scope.

For a learning cluster, the following configuration enables Envoy access logs across the mesh using the built-in `envoy` provider.

Save as `telemetry-logging.yaml`:

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
        - name: envoy
```

Apply it:

```bash
kubectl apply -f telemetry-logging.yaml
```

This is a mesh-wide setting. Before applying it in a shared or production cluster, review existing `Telemetry` resources and coordinate with the mesh administrator.

### 6.2 View proxy logs

Find the pod:

```bash
kubectl get pods -n istio-system
```

View the Istio control-plane logs:

```bash
kubectl logs -n istio-system deployment/istiod
```

To inspect access logs, select an application pod that has an `istio-proxy` container:

```bash
kubectl get pods -n istio-observability
```

Then:

```bash
kubectl logs -n istio-observability <pod-name> -c istio-proxy
```

Replace `<pod-name>` with the actual pod name.

If access logging is enabled, requests through the proxy should produce log entries. Control-plane logs from `istiod` are not the same as application traffic access logs.

### 6.3 Access logs with Loki

Loki is a log aggregation system that stores and queries logs.

A typical pipeline is:

```text
Envoy Proxy Logs
       |
       v
   Log Collector
 (e.g. Grafana Alloy)
       |
       v
      Loki
       |
       v
    Grafana
```

Loki does not automatically receive logs just because it is installed. Configure a log collector to discover Kubernetes pod logs and forward the required container logs to Loki.

For a production environment, define retention, access control, resource limits, and label strategy.

---

## 7. Distributed Tracing

Distributed tracing shows how requests move between services and how much time each operation takes.

A trace consists of spans. Each span represents an operation and may include a start time, duration, service name, attributes, and relationship to other spans.

### 7.1 Trace propagation

Istio can generate proxy spans, but complete end-to-end traces depend on the tracing backend and application behavior.

Applications should forward trace-context headers when making downstream calls. Depending on the tracing setup, headers may include:

- `traceparent`
- `tracestate`
- `x-request-id`
- B3 headers

The required headers depend on the configured tracing system and instrumentation.

If an application does not propagate the relevant context, a trace may appear fragmented rather than showing one continuous request across services.

### 7.2 Common tracing backends

| Tool | Purpose |
|---|---|
| Jaeger | Distributed tracing UI and backend |
| Grafana Tempo | Distributed trace storage and querying |
| OpenTelemetry Collector | Receives, processes, and exports telemetry |
| Zipkin | Distributed tracing backend |

### 7.3 Configure tracing

Tracing requires a compatible backend and a provider configured in Istio's mesh configuration. A `Telemetry` resource selects the provider and sampling behavior.

For example, if a provider named `localtrace` has already been configured in `MeshConfig`, the following resource enables tracing for the mesh with a 10% sampling rate.

Save as `telemetry-tracing.yaml`:

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
    - providers:
        - name: localtrace
      randomSamplingPercentage: 10.00
```

Apply it only after configuring the `localtrace` provider to point to a reachable tracing backend:

```bash
kubectl apply -f telemetry-tracing.yaml
```

**Important:** `localtrace` is an example provider name, not a universally preconfigured tracing backend. Replace it with the exact provider name configured in your cluster.

A 10% sampling rate means Istio randomly selects approximately 10% of requests for trace reporting. Sampling does not guarantee that every request will have a trace.

Official reference: https://istio.io/latest/docs/tasks/observability/distributed-tracing/

---

## 8. Visualizing the Mesh with Kiali

Kiali provides a visual view of services, workloads, traffic, and health within an Istio service mesh.

It helps operators understand:

- Which services communicate with one another
- How much traffic passes between services
- Whether workloads have errors
- Whether Istio configuration is valid
- How traffic is routed through the mesh

### Access Kiali

If Kiali is installed, inspect the service:

```bash
kubectl get svc -n istio-system
```

If the service is named `kiali`, open a local tunnel:

```bash
istioctl dashboard kiali
```

Alternatively:

```bash
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

Then open:

```text
http://localhost:20001
```

The command and port-forward example assume the Kiali service is installed and exposes the expected port. Follow the installation-specific instructions if the service differs.

Kiali depends on telemetry data and commonly integrates with Prometheus. A running Kiali UI alone does not guarantee that useful metrics or traffic graphs are available.

---

## 9. Istio Telemetry API

The Istio Telemetry API provides configuration for how telemetry is generated for workloads.

It can control:

- Metrics behavior
- Access logging
- Distributed tracing
- Sampling and trace reporting
- Workload-specific telemetry settings

### Configuration scope

Telemetry configuration can be applied at different scopes:

1. Mesh-wide — typically in the `istio-system` root configuration namespace.
2. Namespace-wide — applies to workloads in a namespace.
3. Workload-specific — uses a selector to target workloads.

More-specific configuration can override inherited settings. Avoid creating conflicting `Telemetry` resources that target the same workloads.

### Example: enable logging for one workload

Suppose a workload has the label `app: backend` in the `istio-observability` namespace.

Save as `backend-telemetry.yaml`:

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: backend-access-logging
  namespace: istio-observability
spec:
  selector:
    matchLabels:
      app: backend
  accessLogging:
    - providers:
        - name: envoy
```

Apply it:

```bash
kubectl apply -f backend-telemetry.yaml
```

This configures access logging for workloads matching the selector, provided the `envoy` provider is available.

Check the resource:

```bash
kubectl get telemetry -n istio-observability
kubectl describe telemetry backend-access-logging -n istio-observability
```

Official documentation: https://istio.io/latest/docs/tasks/observability/telemetry/

---

## 10. Hands-On Lab

### Objective

Deploy a simple application in Istio and generate traffic that can be used to investigate metrics and proxy logs.

### Prerequisites

- A working Kubernetes cluster
- Istio installed
- `kubectl` and `istioctl`
- Automatic sidecar injection enabled for the lab namespace

### Step 1: Create a namespace

```bash
kubectl create namespace istio-observability
kubectl label namespace istio-observability istio-injection=enabled
```

### Step 2: Deploy the backend and frontend

Create `application.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: istio-observability
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
            - "-text=Hello from the observability backend!"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: istio-observability
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
  namespace: istio-observability
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

Wait for the deployments:

```bash
kubectl rollout status deployment/backend -n istio-observability
kubectl rollout status deployment/frontend -n istio-observability
```

Verify that both pods have an `istio-proxy` container:

```bash
kubectl get pods -n istio-observability
```

### Step 3: Generate traffic

Send several requests from the frontend:

```bash
kubectl exec -n istio-observability deployment/frontend -c curl -- \
  sh -c 'for i in $(seq 1 50); do curl -sS http://backend.istio-observability.svc.cluster.local:5678/; echo; done'
```

You should see the backend response for successful requests.

### Step 4: Inspect proxy logs

Find the backend pod:

```bash
kubectl get pods -n istio-observability -l app=backend
```

Inspect its proxy logs:

```bash
kubectl logs -n istio-observability <backend-pod-name> -c istio-proxy
```

If access logging is not enabled, apply the mesh-wide or workload-specific Telemetry configuration from this module, then generate traffic again.

### Step 5: Inspect Istio configuration

Check proxy synchronization:

```bash
istioctl proxy-status
```

Inspect the backend pod's configuration:

```bash
istioctl proxy-config clusters <backend-pod-name> -n istio-observability
```

Inspect the proxy's listeners:

```bash
istioctl proxy-config listeners <backend-pod-name> -n istio-observability
```

These commands help verify proxy configuration; they are not substitutes for a metrics or tracing backend.

### Step 6: Explore metrics

If Prometheus is installed and configured to scrape the mesh, open its UI and run:

```promql
istio_requests_total
```

Then try:

```promql
sum(rate(istio_requests_total[5m]))
```

If the metric does not appear:

1. Generate more traffic.
2. Confirm the pod has an `istio-proxy` container.
3. Check whether the relevant Prometheus scrape target is healthy.
4. Verify that the metrics endpoint and scrape configuration are correct.
5. Confirm that telemetry filters or custom configuration have not removed the metric.

### Step 7: Explore dashboards

If Grafana and Kiali are installed and configured, explore:

- Request rate
- Error rate
- Request latency
- Backend workload health
- Service-to-service traffic
- Proxy and control-plane health

### Lab challenges

1. Generate at least 100 requests and observe the request counter.
2. Compare metrics before and after increasing the request rate.
3. Enable access logs and identify the response code and request duration.
4. Install or configure a tracing backend and investigate a request across multiple services.
5. Create a Grafana dashboard that displays request rate and HTTP 5xx error rate.
6. Use Kiali to visualize traffic between the frontend and backend.

---

## 11. Troubleshooting

### Problem 1: No Istio metrics appear

Check the pod:

```bash
kubectl get pods -n istio-observability
```

Check whether the proxy container exists:

```bash
kubectl get pod <pod-name> -n istio-observability \
  -o jsonpath='{.spec.containers[*].name}'
```

Check Prometheus targets and confirm that the relevant metrics endpoint is being scraped.

### Problem 2: Access logs are missing

Check the proxy logs:

```bash
kubectl logs -n istio-observability <pod-name> -c istio-proxy
```

Check Telemetry configuration:

```bash
kubectl get telemetry -A
kubectl describe telemetry <telemetry-name> -n <namespace>
```

Confirm that the configured provider exists and that the requests actually pass through the proxy.

### Problem 3: Traces are incomplete

Check the following:

- Is the tracing backend reachable?
- Is the provider correctly configured in `MeshConfig`?
- Is the `Telemetry` resource selecting the intended provider?
- Is trace sampling enabled?
- Are applications forwarding the required trace-context headers?

### Problem 4: Kiali shows no traffic

- Generate traffic between services.
- Confirm that Prometheus is available and contains Istio metrics.
- Check Kiali's data-source configuration.
- Verify that the selected namespace and time range include the test traffic.

### Problem 5: High request latency

Investigate:

- Request duration metrics
- Distributed traces
- Proxy access logs
- Application logs
- Resource saturation and Kubernetes events
- Retries, timeouts, and connection-pool limits

Remember that a slow request may originate in the application, a downstream dependency, or the network path. Use multiple telemetry signals to narrow down the cause.

---

## 12. Best Practices

- Monitor request rate, error rate, latency, and resource saturation.
- Configure alerts for meaningful service-level objectives.
- Use trace sampling appropriate to traffic volume and operational needs.
- Ensure applications propagate trace context across service calls.
- Centralize logs and configure sensible retention policies.
- Avoid high-cardinality metric labels that create excessive time-series data.
- Avoid placing secrets or sensitive personal data in logs and trace attributes.
- Use workload-specific telemetry settings when mesh-wide settings are too broad.
- Monitor Istiod and Envoy proxies as well as application workloads.
- Protect Prometheus, Grafana, Kiali, and tracing interfaces with appropriate access controls.
- Treat sample addons as learning tools; harden and scale monitoring components for production.

---

## 13. Interview Questions

### Beginner

**1. What is observability?**

Observability is the ability to understand a system's internal state using telemetry such as metrics, logs, and traces.

**2. What are the three pillars of observability?**

Metrics, logs, and distributed traces.

**3. What is the role of Envoy in Istio observability?**

Envoy proxies generate telemetry about traffic passing through them, including request metrics and, when configured, access logs and trace spans.

**4. What is Prometheus?**

Prometheus is a monitoring system that collects and stores time-series metrics and supports queries through PromQL.

**5. What is Grafana?**

Grafana visualizes data from sources such as Prometheus and provides dashboards and alerting capabilities.

### Intermediate

**6. What is distributed tracing?**

Distributed tracing follows a request across multiple services and helps identify slow operations and dependency failures.

**7. What is Kiali?**

Kiali visualizes service relationships, traffic, and health in an Istio service mesh.

**8. What is the Istio Telemetry API?**

It is an Istio configuration API for controlling how telemetry such as metrics, access logs, and traces is generated for workloads.

**9. Why might a trace be incomplete?**

The application may not propagate trace-context headers, the sampling rate may exclude the request, or the tracing backend/provider may be misconfigured.

**10. Does installing Istio automatically install Prometheus, Grafana, Jaeger, and Kiali?**

Not necessarily. The core Istio installation and optional observability addons are separate. Their availability depends on the installation method and configuration.

### Advanced

**11. How would you troubleshoot an increase in HTTP 5xx errors?**

Check error-rate metrics, filter logs by response code, inspect traces for failed dependencies, and verify routing, application health, and proxy configuration.

**12. Why is high metric cardinality a problem?**

A large number of unique label combinations creates many time series, increasing memory, storage, and query costs.

**13. What is trace sampling?**

Trace sampling determines which requests are selected for trace reporting. It reduces telemetry overhead but means not every request will necessarily have a trace.

**14. What is the difference between application logs and Envoy access logs?**

Application logs describe events inside the application. Envoy access logs describe traffic observed and processed by the proxy.

**15. How would you build an observability stack for Istio in production?**

I would configure metrics collection and alerting, centralized access and application logs, distributed tracing with context propagation, mesh visualization, retention policies, access controls, and resource sizing. I would also define service-level indicators and validate the stack under realistic traffic.

---

## 14. Next Module

Continue to:

[**10. Ambient Mesh →**](../10-ambient-mesh/README.md)

## 📖 Official Documentation

- [Istio Observability](https://istio.io/latest/docs/concepts/observability/)
- [Observability Tasks](https://istio.io/latest/docs/tasks/observability/)
- [Telemetry API](https://istio.io/latest/docs/tasks/observability/telemetry/)
- [Envoy Access Logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/)
- [Distributed Tracing](https://istio.io/latest/docs/tasks/observability/distributed-tracing/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Kiali Documentation](https://kiali.io/docs/)

---

**Previous:** [08. Istio Security and Zero Trust](../08-istio-security-zero-trust/README.md)  
**Next:** [10. Ambient Mesh](../10-ambient-mesh/README.md)
