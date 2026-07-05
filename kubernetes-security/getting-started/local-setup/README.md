---
foundation:
  topic: Kubernetes Security
  group: Getting Started
  slug: local-setup
  image: ../assets/getting-started.svg
  order: 10
  title: "Local Setup"
  description: "Install the tools you need to run a local Kubernetes cluster for the hands on exercises."
  publish: true
---

# Local Setup

This foundation uses a local Kubernetes cluster so you can test security controls without touching a shared environment. The cluster should be disposable: if something breaks, you should be able to delete it and create a fresh one in a few minutes.

The examples below use `kind` because it runs Kubernetes nodes as containers and is easy to reset. You can use another local option, but the commands in this foundation assume `kubectl` and a working kubeconfig.

## What You Need

Install these tools:

- **Docker Desktop**, Colima, Rancher Desktop, or another container runtime that can run Docker-compatible containers.
- **kubectl**, the Kubernetes command-line client.
- **kind**, for creating a local Kubernetes cluster.
- **jq**, useful for reading JSON output from Kubernetes commands.
- **openssl**, useful when exploring certificates and tokens.

On macOS with Homebrew:

```bash
brew install kubectl kind jq openssl
```

Check that the tools are available:

```bash
docker version
kubectl version --client
kind version
jq --version
openssl version
```

If `docker version` cannot connect to a daemon, start your container runtime before continuing.

## Create a Working Directory

Keep local experiment files in one place:

```bash
mkdir -p ~/k8s-security-lab
cd ~/k8s-security-lab
```

This is where you can store temporary manifests, command output, and notes while moving through the foundation.

## Choose a Cluster Name

Use a consistent cluster name so cleanup commands are predictable:

```bash
export K8S_SECURITY_CLUSTER=k8s-security-lab
```

You can add that export to your shell profile if you want, but it is not required.

## Local Cluster Expectations

The lab cluster is for learning, not production simulation. That matters because a local cluster has tradeoffs:

- It usually runs with simplified networking.
- It may not expose the same managed-control-plane settings as a cloud cluster.
- It may not include your real ingress controller, CNI, cloud IAM integration, or logging stack.
- It is still good enough to learn RBAC, service accounts, admission behavior, pod security settings, secrets, workload hardening, and many network policy concepts.

When a topic depends on cloud-provider behavior or production-grade observability, the page will call that out.

## Reset Strategy

Security learning works best when you are not afraid to break things. Use this mental model:

- If an exercise changes a namespace, delete the namespace.
- If an exercise changes cluster-wide policy, remove the policy or recreate the cluster.
- If the cluster behaves strangely, delete it and start again.

The next page creates the disposable cluster. Do not use a production kubeconfig for these exercises.
