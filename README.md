<div align="center">

# Istio Zero to Hero 🚀

<img src="assets/istio.png" alt="Istio Zero to Hero" width="320"/>

### A Complete Hands-On Guide to Istio Service Mesh — From Fundamentals to Production

Learn Istio from scratch and progress to traffic management, zero-trust security, observability, ambient mesh, and production deployments on Kubernetes and Amazon EKS.

[![Istio][badge-istio]][istio-docs]
[![Kubernetes][badge-k8s]][k8s]
[![Envoy][badge-envoy]][envoy]
[![AWS EKS][badge-eks]][eks]
[![Terraform][badge-tf]][terraform]
[![Helm][badge-helm]][helm]

[![License: MIT][badge-license]](./LICENSE)
[![PRs Welcome][badge-prs]](#-contributing)
[![Modules][badge-modules]](#-table-of-contents)
[![Level][badge-level]](#-table-of-contents)
[![Hands-on Labs][badge-labs]](#-table-of-contents)

[📖 Start Learning](#-table-of-contents) •
[🎯 Objectives](#-learning-objectives) •
[🛠️ Prerequisites](#️-prerequisites) •
[🧰 Tools](#-tools-and-technologies) •
[🤝 Contribute](#-contributing)

</div>

---

## 📌 About This Repository

This repository provides structured notes, practical examples, configuration manifests, hands-on labs, troubleshooting guides, and real-world projects to help you master Istio.

### What Is Istio?

Istio is an open-source service mesh that provides a consistent way to **connect, secure, manage, and observe** communication between microservices.

It enables traffic routing, load balancing, mutual TLS (mTLS), access control, retries, fault injection, and distributed observability, all without implementing these features individually in every application.

Istio integrates with Kubernetes and supports both **sidecar-based** and **ambient** service mesh architectures.

```
          ┌───────────────────────────────────────────┐
          │          Control Plane (Istiod)           │
          │   config  •  certificates  •  discovery   │
          └───────────────────┬───────────────────────┘
                              │ xDS / mTLS certs
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
 ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
 │ App + Envoy │◄─────►│ App + Envoy │◄─────►│ App + Envoy │
 └─────────────┘  mTLS └─────────────┘  mTLS └─────────────┘
              Data Plane (service-to-service traffic)
```

---

## 🎯 Learning Objectives

By completing this repository, you will learn how to:

| # | Objective |
|---|---|
| 1 | Understand service mesh architecture and Istio components |
| 2 | Install and configure Istio on Kubernetes |
| 3 | Manage application traffic using Istio networking resources |
| 4 | Implement canary deployments, traffic splitting, and fault tolerance |
| 5 | Secure service-to-service communication with mTLS and authorization policies |
| 6 | Integrate Istio with Prometheus, Grafana, OpenTelemetry, Tempo, and Kiali |
| 7 | Understand and implement Istio ambient mesh |
| 8 | Troubleshoot real-world service mesh issues |
| 9 | Deploy Istio on Amazon EKS |
| 10 | Implement zero-trust networking and production-ready architectures |

---

## 🗺️ Learning Path

```
 🟢 BEGINNER            🟡 INTERMEDIATE            🔴 ADVANCED             ☁️ PRODUCTION
 ───────────            ───────────────            ───────────             ─────────────
 01 Fundamentals   ──►  05 Traffic Mgmt      ──►   10 Ambient Mesh   ──►   15 Istio on EKS
 02 Architecture        06 Canary Deploys          11 Gateways             16 DevSecOps
 03 Installation        07 Resilience              12 K8s Integration      17 Projects
 04 First App           08 Security / Zero Trust   13 Troubleshooting      18 Interview Prep
                        09 Observability           14 Advanced Arch
```

---

## 📚 Table of Contents

### 🟢 Beginner Level

| Module | Topic | Description |
|:---:|---|---|
| 01 | [Service Mesh Fundamentals](./01-service-mesh-fundamentals/README.md) | Understand service meshes, their purpose, benefits, and use cases. |
| 02 | [Istio Architecture](./02-istio-architecture/README.md) | Learn about Istiod, Envoy, control plane, and data plane. |
| 03 | [Installing Istio](./03-installing-istio/README.md) | Install Istio using istioctl and Helm on Kubernetes. |
| 04 | [Your First Istio Application](./04-first-istio-application/README.md) | Deploy an application, enable proxy injection, and explore traffic flow. |

### 🟡 Intermediate Level

| Module | Topic | Description |
|:---:|---|---|
| 05 | [Istio Traffic Management](./05-traffic-management/README.md) | Learn VirtualService, DestinationRule, gateways, routing, and load balancing. |
| 06 | [Canary and Advanced Deployments](./06-canary-deployments/README.md) | Implement canary releases, blue-green deployments, and traffic shifting. |
| 07 | [Resilience and Fault Tolerance](./07-resilience-fault-tolerance/README.md) | Configure retries, timeouts, circuit breaking, and fault injection. |
| 08 | [Istio Security and Zero Trust](./08-istio-security-zero-trust/README.md) | Implement mTLS, workload identity, JWT authentication, and authorization. |
| 09 | [Istio Observability](./09-observability/README.md) | Integrate Prometheus, Grafana, Kiali, OpenTelemetry, and distributed tracing. |

### 🔴 Advanced Level

| Module | Topic | Description |
|:---:|---|---|
| 10 | [Istio Ambient Mesh](./10-ambient-mesh/README.md) | Explore sidecar-less architecture, ztunnel, HBONE, and waypoint proxies. |
| 11 | [Gateways and External Traffic](./11-gateways-external-traffic/README.md) | Configure ingress, egress, TLS, and external service access. |
| 12 | [Istio and Kubernetes Integration](./12-kubernetes-integration/README.md) | Integrate Istio with Kubernetes networking, autoscaling, Helm, and Argo CD. |
| 13 | [Operations and Troubleshooting](./13-operations-troubleshooting/README.md) | Diagnose routing, proxy, certificate, and connectivity problems. |
| 14 | [Advanced Istio Architecture](./14-advanced-architecture/README.md) | Learn multi-cluster meshes, cross-cluster communication, and production operations. |

### ☁️ Cloud Security and Production

| Module | Topic | Description |
|:---:|---|---|
| 15 | [Istio on AWS EKS](./15-istio-on-eks/README.md) | Deploy and operate Istio on Amazon EKS with Terraform and GitOps. |
| 16 | [Service Mesh Security and DevSecOps](./16-security-devsecops/README.md) | Apply zero-trust principles, least privilege, and security monitoring. |
| 17 | [Production-Ready Projects](./17-projects/README.md) | Build end-to-end projects covering traffic management, security, and observability. |
| 18 | [Interview Preparation](./18-interview-preparation/README.md) | Prepare for Istio interviews with technical questions and production scenarios. |

---

## 🛠️ Prerequisites

Before starting, you should be comfortable with:

| Area | Topics |
|---|---|
| **Fundamentals** | Linux and command-line tools, Git and GitHub |
| **Containers** | Docker and containerization |
| **Kubernetes** | Architecture, workloads, Services, Deployments, Pods, namespaces, kubectl, troubleshooting |
| **Configuration** | YAML and Kubernetes manifests |
| **Networking** | Basic networking, DNS, HTTP, and TLS |

> 💡 Familiarity with **Helm, Terraform, Argo CD, and Prometheus** is helpful for the advanced modules.

---

## 🧰 Tools and Technologies

| Category | Tools |
|---|---|
| Service Mesh | Istio, Envoy Proxy |
| Container Platform | Docker, Kubernetes |
| Package Management | Helm |
| GitOps | Argo CD |
| Infrastructure as Code | Terraform |
| Monitoring | Prometheus, Grafana |
| Logging | Loki, Alloy |
| Distributed Tracing | OpenTelemetry, Grafana Tempo |
| Service Visualization | Kiali |
| Cloud Platform | AWS, Amazon EKS |
| Security | mTLS, JWT, AuthorizationPolicy, Kyverno, Falco |

---

## 📖 Official Resources

| Resource | Link |
|---|---|
| 📘 Documentation | [Istio Documentation](https://istio.io/latest/docs/) |
| 🚦 Getting Started | [Istio Getting Started](https://istio.io/latest/docs/setup/getting-started/) |
| 🔀 Traffic Management | [Concepts](https://istio.io/latest/docs/concepts/traffic-management/) |
| 🔐 Security | [Concepts](https://istio.io/latest/docs/concepts/security/) |
| 📊 Observability | [Tasks](https://istio.io/latest/docs/tasks/observability/) |
| 🌐 Ambient Mesh | [Ambient Docs](https://istio.io/latest/docs/ambient/) |
| 💻 Source Code | [Istio GitHub Repository](https://github.com/istio/istio) |

---

## 🤝 Contributing

Contributions are welcome! If you find an issue, have a suggestion, or want to improve a lab:

1. **Fork** this repository.
2. **Create** a feature branch.
3. **Make** your changes.
4. **Submit** a pull request with a clear description.

Please ensure that examples are tested and compatible with the documented Istio version.

---

## 📄 License

This project is licensed under the **MIT License**. See the [LICENSE](./LICENSE) file for details.

---

<div align="center">

### Happy Learning! Master Istio one module at a time. 🚀

⭐ If you find this repository helpful, consider giving it a star!

</div>

<!-- Badge references -->
[badge-istio]: https://img.shields.io/badge/Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white
[badge-k8s]: https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white
[badge-envoy]: https://img.shields.io/badge/Envoy-AC6199?style=for-the-badge&logo=envoyproxy&logoColor=white
[badge-eks]: https://img.shields.io/badge/Amazon_EKS-FF9900?style=for-the-badge&logo=amazoneks&logoColor=white
[badge-tf]: https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white
[badge-helm]: https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white
[badge-license]: https://img.shields.io/badge/License-MIT-green?style=flat-square
[badge-prs]: https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square
[badge-modules]: https://img.shields.io/badge/Modules-18-blue?style=flat-square
[badge-level]: https://img.shields.io/badge/Level-Beginner_to_Advanced-orange?style=flat-square
[badge-labs]: https://img.shields.io/badge/Hands--on-Labs-red?style=flat-square

<!-- Link references -->
[istio-docs]: https://istio.io/latest/docs/
[k8s]: https://kubernetes.io/
[envoy]: https://www.envoyproxy.io/
[eks]: https://aws.amazon.com/eks/
[terraform]: https://www.terraform.io/
[helm]: https://helm.sh/
