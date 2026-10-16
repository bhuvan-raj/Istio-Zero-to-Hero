# Module 03: Installing Istio

Learn how to install Istio on a Kubernetes cluster using `istioctl`, configure automatic sidecar injection, verify the installation, and troubleshoot common issues.

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [1. Understand Istio Installation](#1-understand-istio-installation)
- [2. Verify Your Kubernetes Cluster](#2-verify-your-kubernetes-cluster)
- [3. Download Istio](#3-download-istio)
- [4. Install Istio](#4-install-istio)
- [5. Verify the Installation](#5-verify-the-installation)
- [6. Enable Automatic Sidecar Injection](#6-enable-automatic-sidecar-injection)
- [7. Deploy a Test Application](#7-deploy-a-test-application)
- [8. Verify Sidecar Injection](#8-verify-sidecar-injection)
- [9. Explore Istio with istioctl](#9-explore-istio-with-istioctl)
- [10. Troubleshooting](#10-troubleshooting)
- [11. Uninstall Istio](#11-uninstall-istio)
- [Key Takeaways](#key-takeaways)
- [Interview Questions](#interview-questions)
- [Next Module](#next-module)
- [Official References](#official-references)

## Learning Objectives

By the end of this module, you will be able to:

- Install Istio on a Kubernetes cluster.
- Understand Istio installation profiles.
- Use `istioctl` to manage and inspect Istio.
- Enable automatic Envoy sidecar injection.
- Verify the Istio control plane and data plane.
- Troubleshoot common installation problems.

## Prerequisites

Before starting, ensure you have:

- A working Kubernetes cluster.
- `kubectl` installed and configured.
- Cluster permissions to create namespaces, deployments, and Istio resources.
- `curl`, `tar`, and a supported Linux or macOS environment for the download commands below.
- Basic knowledge of Kubernetes Deployments, Services, Pods, and namespaces.

You can use Minikube, Kind, a managed Kubernetes cluster, or another supported Kubernetes environment.

## 1. Understand Istio Installation

Istio can be installed using different methods, including `istioctl` and Helm.

For this module, we will use **istioctl**, the official Istio command-line tool.

The main components installed in a traditional sidecar-based setup include:

| Component | Purpose |
|---|---|
| Istiod | Control plane responsible for configuration, service discovery, and workload identity |
| Envoy proxy | Handles traffic for application workloads |
| Injection webhook | Automatically injects Envoy sidecars into eligible new Pods |
| Istio gateway | Handles configured ingress or egress traffic when the gateway component is installed |

### Installation profiles

Istio provides predefined profiles for different use cases.

| Profile | Purpose |
|---|---|
| `default` | General-purpose starting point, including production-oriented defaults |
| `demo` | Learning, demonstrations, and experimenting with Istio features |
| `minimal` | Minimal control-plane installation |
| `ambient` | Installs components for Istio's ambient data plane |
| `empty` | Base profile with no components selected |

For this learning module, we will use the `demo` profile. It is intended for evaluation and learning, not as a production configuration.

## 2. Verify Your Kubernetes Cluster

Check whether Kubernetes is accessible:

```bash
kubectl cluster-info
```

Check the active context:

```bash
kubectl config current-context
```

List the nodes:

```bash
kubectl get nodes
```

Check whether the nodes are ready:

```bash
kubectl get nodes -o wide
```

**Expected result:** Your cluster should be reachable, and the nodes should be in the `Ready` state.

If these commands fail, fix the Kubernetes connectivity or permissions before installing Istio.

## 3. Download Istio

Download and extract the latest stable Istio release for Linux or macOS using the official download script:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Move into the extracted Istio directory. Replace the version below with the directory created by the download command.

```bash
cd istio-<version>
```

For example, if the extracted directory is named `istio-1.x.y`, use that exact directory name.

Add `istioctl` to your current shell's `PATH`:

```bash
export PATH="$PWD/bin:$PATH"
```

Verify the client version:

```bash
istioctl version --remote=false
```

You can also check the available commands:

```bash
istioctl --help
```

**Important:** The `PATH` change above applies only to the current shell session. If you open another terminal, you may need to add the `bin` directory to your `PATH` again.

## 4. Install Istio

Install Istio using the `demo` profile:

```bash
istioctl install --set profile=demo -y
```

This installs the components selected by the demo profile into the Kubernetes cluster associated with your current `kubectl` context.

Wait for the installation to finish. You should see output indicating that the installation completed successfully.

### What happens during installation?

1. `istioctl` generates and applies the required Kubernetes resources.
2. Istiod is deployed as the Istio control plane.
3. The selected gateway and supporting components are installed.
4. Istio configures the resources needed for sidecar injection and validation.

Check the Istio namespace:

```bash
kubectl get namespace istio-system
```

List the resources in that namespace:

```bash
kubectl get all -n istio-system
```

## 5. Verify the Installation

### Check Istio Pods

```bash
kubectl get pods -n istio-system
```

### Check Istio Services

```bash
kubectl get services -n istio-system
```

### Check Istio version and cluster status

```bash
istioctl version
```

### Analyze the cluster configuration

```bash
istioctl analyze -A
```

This command analyzes Istio configuration and reports potential problems. Warnings do not necessarily mean the installation has failed; review each finding in context.

**Expected result:** Istiod and the components selected by your installation profile should be running and ready.

## 6. Enable Automatic Sidecar Injection

Installing Istio does not automatically mean every existing application Pod will receive an Envoy sidecar.

To enable automatic injection for a namespace, label it.

For this lab, we will use the `default` namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

Verify the label:

```bash
kubectl get namespace default --show-labels
```

You should see:

```text
istio-injection=enabled
```

### What does this label do?

When a new eligible Pod is created in this namespace, Istio's injection webhook can modify its Pod specification to include an Envoy sidecar.

Remember:

- The label applies to the namespace, not directly to a Deployment.
- Existing Pods are not automatically modified just because the label was added.
- Recreate existing application Pods to have them considered for injection.
- Injection can also be controlled through other Istio configuration options.

## 7. Deploy a Test Application

We will use the HTTPBin sample included in the downloaded Istio package.

Make sure you are still in the extracted Istio directory.

Deploy the sample:

```bash
kubectl apply -f samples/httpbin/httpbin.yaml
```

Check the Deployment and Service:

```bash
kubectl get deployments,services
```

Check the Pods:

```bash
kubectl get pods
```

Wait for the HTTPBin Pod to become ready:

```bash
kubectl rollout status deployment/httpbin
```

If your cluster takes longer to start the application, inspect the Pod status and events before proceeding.

## 8. Verify Sidecar Injection

List the HTTPBin Pod:

```bash
kubectl get pods -l app=httpbin
```

Check the containers in the Pod:

```bash
kubectl get pods -l app=httpbin \
  -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

The output should include both the application container and `istio-proxy`.

You can also inspect the Pod details:

```bash
kubectl describe pod -l app=httpbin
```

### Verify the sidecar with istioctl

Find the HTTPBin Pod name:

```bash
kubectl get pods -l app=httpbin
```

Then inspect the workload:

```bash
istioctl x describe pod <httpbin-pod-name>
```

Replace `<httpbin-pod-name>` with the actual Pod name.

**What you have verified:** The application Pod contains an Envoy sidecar, which is the data-plane proxy used by Istio in sidecar mode.

## 9. Explore Istio with istioctl

Here are useful commands for inspecting your installation.

### Check the control plane and proxies

```bash
istioctl version
```

```bash
istioctl proxy-status
```

`istioctl proxy-status` shows the connection and synchronization status of discovered Envoy proxies with the Istio control plane.

### Inspect Envoy listeners

```bash
istioctl proxy-config listeners <httpbin-pod-name>
```

### Inspect Envoy clusters

```bash
istioctl proxy-config clusters <httpbin-pod-name>
```

### Inspect Envoy routes

```bash
istioctl proxy-config routes <httpbin-pod-name>
```

### Inspect Envoy endpoints

```bash
istioctl proxy-config endpoints <httpbin-pod-name>
```

These commands help you investigate how Istio configures traffic routing and service connectivity.

## 10. Troubleshooting

### Problem 1: Istio Pods are not ready

Check the Pod status:

```bash
kubectl get pods -n istio-system
```

Inspect a failing Pod:

```bash
kubectl describe pod <pod-name> -n istio-system
```

Check its logs:

```bash
kubectl logs <pod-name> -n istio-system
```

Look for image-pull failures, scheduling issues, insufficient resources, or configuration errors.

### Problem 2: The Envoy sidecar is missing

Verify the namespace label:

```bash
kubectl get namespace default --show-labels
```

Check whether the application Pod was created after injection was enabled.

If necessary, recreate the sample Deployment's Pods:

```bash
kubectl rollout restart deployment/httpbin
```

Then verify the containers again:

```bash
kubectl get pods -l app=httpbin \
  -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

### Problem 3: istioctl cannot connect to the cluster

Check the active Kubernetes context:

```bash
kubectl config current-context
```

Check cluster connectivity:

```bash
kubectl get nodes
```

Ensure that `kubectl` is using the intended cluster and that your account has the necessary permissions.

### Problem 4: The application Pod is stuck

Inspect its status and events:

```bash
kubectl describe pod <httpbin-pod-name>
```

Check the application container logs:

```bash
kubectl logs <httpbin-pod-name> -c httpbin
```

If the container name differs, use the name shown by the Pod inspection command.

### Problem 5: Istio reports configuration warnings

Run:

```bash
istioctl analyze -A
```

Review the reported resource names and recommendations. Resolve the underlying configuration problem rather than suppressing warnings without understanding them.

## 11. Uninstall Istio

> **Warning:** Run these commands only in a disposable learning cluster or when you intentionally want to remove Istio. Removing Istio can affect applications that depend on its traffic management or security features.

Uninstall Istio:

```bash
istioctl uninstall --purge
```

Remove the control-plane namespace if it remains and you no longer need it:

```bash
kubectl delete namespace istio-system
```

Remove the injection label from the default namespace:

```bash
kubectl label namespace default istio-injection-
```

Remove the test application:

```bash
kubectl delete -f samples/httpbin/httpbin.yaml
```

The uninstall command removes Istio resources, including cluster-scoped resources that may be shared with other Istio control planes. Do not use `--purge` on a shared cluster without reviewing its impact.

## Key Takeaways

- `istioctl` is the official CLI for installing and managing Istio.
- Installation profiles provide predefined configurations for different use cases.
- The `demo` profile is useful for learning, while production installations need deliberate configuration and sizing.
- Istiod is the control plane; Envoy proxies handle application traffic in sidecar mode.
- Namespace labels can enable automatic sidecar injection for new eligible Pods.
- `istioctl proxy-status`, `istioctl proxy-config`, and `istioctl analyze` are useful troubleshooting tools.

## Interview Questions

1. What is `istioctl`?
2. What is the difference between the `default` and `demo` installation profiles?
3. How do you install Istio on Kubernetes?
4. What is the purpose of the `istio-system` namespace?
5. How do you enable automatic sidecar injection?
6. Will enabling injection modify existing Pods automatically?
7. How do you verify that an Envoy sidecar has been injected?
8. What does `istioctl proxy-status` show?
9. How would you troubleshoot an Istio Pod that is not ready?
10. How do you uninstall Istio safely?

## Next Module

Continue to **[Module 04: Your First Istio Application](../04-first-istio-application/README.md)** to deploy an application and begin exploring service-mesh behavior.

## Official References

- [Istio Installation Guides](https://istio.io/latest/docs/setup/install/)
- [Install Istio with istioctl](https://istio.io/latest/docs/setup/install/istioctl/)
- [Istio Getting Started Guide](https://istio.io/latest/docs/setup/getting-started/)
- [Istio Configuration Profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)
- [istioctl Command Reference](https://istio.io/latest/docs/reference/commands/istioctl/)

---

**Repository:** [Istio-Zero-to-Hero](https://github.com/bhuvan-raj/Istio-Zero-to-Hero)

**Module:** 03 — Installing Istio
