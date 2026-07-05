---
foundation:
  topic: Kubernetes Security
  group: Overview
  slug: overview
  image: assets/overview.svg
  order: 0
  title: "Overview"
  description: "What Kubernetes security covers, the cluster attack surface, and how this foundation is structured."
  publish: true
---

# Overview

Kubernetes security is about controlling four things:

- **Who** can use the cluster.
- **What** workloads are allowed to run.
- **Where** those workloads can reach.
- **How quickly** you can detect and respond when something goes wrong.

Kubernetes is not a single server to harden. It is an API server, scheduler, worker nodes, identities, permissions, networks, secrets, workloads, and container images working together. A weak setting in any one of those areas can become a path into the rest of the cluster.

## What This Covers

This foundation covers the main parts of the Kubernetes security surface:

- **Identity and authorization:** who is calling the API and what they can do.
- **Admission control:** what is allowed into the cluster before it is created.
- **Cluster hardening:** how the control plane and nodes are configured.
- **Network security:** which pods and services can talk to each other.
- **Secrets management:** how credentials are stored and exposed.
- **Workload security:** how pods, containers, Linux capabilities, and filesystems are constrained.
- **Supply chain security:** whether the images you run are trusted.
- **Detection and response:** how you investigate suspicious activity.

The goal is layered control. No single setting secures a cluster by itself.

## How To Use This Foundation

Start with **Getting Started** if you want a disposable local cluster for hands-on practice. Then read **Identity** and **Authorization** before the other sections; most Kubernetes security decisions depend on understanding who is making a request and what that identity can do.

After that:

- For application deployment security, focus on Admission Control, Workload Security, Network Security, and Secrets Management.
- For cluster operations, focus on Cluster Hardening, Detection & Response, and the Summary checklist.
- For CI/CD and platform controls, focus on Supply Chain Security and Admission Control.

## A Practical Baseline

Start with these habits:

1. Use short-lived, purpose-specific identities.
2. Grant the smallest RBAC permissions that still let work happen.
3. Avoid privileged pods, host namespaces, hostPath mounts, and unnecessary Linux capabilities.
4. Apply pod security standards at the namespace level.
5. Use NetworkPolicies for important namespaces.
6. Keep secrets out of images, logs, and plain manifests where possible.
7. Scan and verify images before they run.
8. Keep audit and runtime signals available for investigation.

The examples are intentionally local and disposable so you can learn the mechanics before applying them to a shared or production cluster.
