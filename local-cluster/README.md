# Local Cluster

Different ways to spin up a local Kubernetes cluster for development, testing, and learning. Each subfolder covers one tool: what it is, when to use it, and how to create and tear down a cluster.

| Tool | What it is | Best for |
|------|------------|----------|
| [kind](./kind/) | Kubernetes IN Docker (nodes run as containers) | CI pipelines, multi-node testing |
| [minikube](./minikube/) | Single/multi-node cluster via a VM or container driver | Beginners, rich addon ecosystem |
| [k3d](./k3d/) | Runs k3s inside Docker | Fast, low-resource, multi-cluster |
| [k3s](./k3s/) | Lightweight certified Kubernetes, single binary on the host | Edge, IoT, low-resource hosts |
| [docker-desktop](./docker-desktop/) | Built-in Kubernetes toggle in Docker Desktop | Simplest start on Mac/Windows |
| [rancher-desktop](./rancher-desktop/) | Desktop app with k3s-based Kubernetes and a container runtime | GUI-driven local dev |
| [microk8s](./microk8s/) | Canonical's single-package Kubernetes (snap) | Linux, single-command install |
| [colima](./colima/) | Container runtime on macOS/Linux with optional Kubernetes | Lightweight Docker Desktop alternative |
| [kubeadm](./kubeadm/) | Official tool to bootstrap a cluster from scratch | Closest to a production setup |

## Choosing one

- **Just want a cluster fast:** kind or k3d (both Docker-based and quick to reset).
- **On a Mac or Windows laptop:** Docker Desktop or Rancher Desktop for a GUI, or colima for a light CLI option.
- **Learning the internals / production-like:** kubeadm.
- **Low-resource or edge:** k3s or microk8s.
