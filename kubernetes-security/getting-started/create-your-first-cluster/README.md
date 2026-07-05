---
foundation:
  topic: Kubernetes Security
  group: Getting Started
  slug: create-your-first-cluster
  image: ../assets/getting-started.svg
  order: 11
  title: "Create Your First Cluster"
  description: "Spin up a disposable kind cluster you can safely break and rebuild while learning."
  publish: true
---

# Create Your First Cluster

This page creates a disposable `kind` cluster for the hands-on Kubernetes security exercises.

Before running these commands, make sure your container runtime is running and that `kubectl` and `kind` are installed.

## Create the Cluster

Set a cluster name:

```bash
export K8S_SECURITY_CLUSTER=k8s-security-lab
```

Create the cluster:

```bash
kind create cluster --name "$K8S_SECURITY_CLUSTER"
```

`kind` creates a kubeconfig context named `kind-k8s-security-lab`. Switch to it explicitly:

```bash
kubectl config use-context "kind-$K8S_SECURITY_CLUSTER"
```

Confirm the context before doing anything else:

```bash
kubectl config current-context
```

Expected output:

```text
kind-k8s-security-lab
```

## Check the Nodes

List the nodes:

```bash
kubectl get nodes -o wide
```

A default `kind` cluster usually has one control-plane node. That is enough for most examples in this foundation.

You can also check the core system pods:

```bash
kubectl get pods -n kube-system
```

Wait until the system pods are `Running` or `Completed` before continuing.

## Create a Practice Namespace

Use a separate namespace for basic experiments:

```bash
kubectl create namespace lab
```

Set it as the default namespace for the current context:

```bash
kubectl config set-context --current --namespace=lab
```

Confirm it:

```bash
kubectl config view --minify --output 'jsonpath={..namespace}'; echo
```

Expected output:

```text
lab
```

## Deploy a Test Workload

Create a small workload:

```bash
kubectl create deployment hello --image=nginx:1.27
kubectl rollout status deployment/hello
```

List the pod:

```bash
kubectl get pods
```

This gives you a simple target for later exercises around service accounts, network access, pod configuration, and runtime behavior.

## Clean Up the Test Workload

Remove the deployment when you are done with the initial test:

```bash
kubectl delete deployment hello
```

Keep the `lab` namespace. Later pages can reuse it.

## Delete the Cluster

When you want to start over:

```bash
kind delete cluster --name "$K8S_SECURITY_CLUSTER"
```

The ability to reset quickly is part of the lab design. You should be comfortable destroying and recreating this cluster throughout the foundation.
