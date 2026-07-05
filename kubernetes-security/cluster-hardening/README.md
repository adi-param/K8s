---
foundation:
  topic: Kubernetes Security
  group: Cluster Hardening
  slug: overview
  image: assets/cluster-hardening.svg
  order: 90
  title: "Overview"
  description: "Securing the control plane components that the rest of the cluster trusts."
  publish: true
---

# Overview

Everything covered so far (RBAC, admission policies, pod hardening) assumes one thing: **the control plane itself is trustworthy**. Cluster hardening is about making that assumption true.

The control plane is the cluster's crown jewels. Whoever controls it controls every workload, every secret, and every node:

```text
kubectl -> API server -> etcd
                |
                v
             kubelet (on every node)
```

- **API server:** the front door. Every request, whether human, CI pipeline, or controller, goes through it. A misconfigured API server bypasses every policy behind it.
- **etcd:** the database. Every object and every Secret lives here. Read access to etcd equals read access to everything, no RBAC involved.
- **kubelet:** the node agent. It can start containers, read pod logs, and exec into workloads on its node. Its API is a second, less-watched door into the cluster.

## Why This Layer Is Different

Workload security limits what a compromised *container* can do. Cluster hardening limits what a compromised *credential, network position, or node* can do. The failure modes are worse:

| Weakness | Consequence |
| --- | --- |
| API server reachable from the internet with weak auth | Full cluster takeover from anywhere |
| Unencrypted etcd, or a leaked etcd backup | Every Secret in the cluster, offline, in plaintext |
| Kubelet API with anonymous access | Exec into any pod on that node without touching RBAC |
| Overly broad admission plugin or authorization mode | Every other control silently weakened |

None of these are exotic. They are default-adjacent configurations that early Kubernetes versions actually shipped with, and that benchmarks still catch in real clusters today.

## What This Section Covers

1. **API Server:** the flags that decide who gets in and what gets recorded: anonymous auth, authorization modes, admission plugins, audit logging, and network exposure.
2. **etcd:** proving to yourself that Secrets sit in etcd in plaintext by default, then fixing it with encryption at rest, TLS, and careful backup handling.
3. **Kubelet:** closing the node agent's anonymous and read-only ports and making it authenticate and authorize like everything else.
4. **CIS Benchmark:** measuring all of the above automatically with kube-bench instead of eyeballing flags.

## Managed Clusters Shift, Not Remove, The Work

On EKS, GKE, or AKS, the provider runs the API server and etcd for you. That genuinely removes classes of risk, because you cannot forget to encrypt a disk you do not manage. But responsibility shifts rather than disappears:

- **You still own** API endpoint exposure (public vs private), authentication integration, audit log collection, node configuration, and everything from the kubelet down.
- **The provider owns** control plane patching, etcd operations, and physical security.
- **The benchmark changes:** CIS publishes provider-specific benchmarks (CIS EKS, CIS GKE) because half the standard checks are the provider's job.

The pages in this section use the local `kind` lab cluster, where you run the whole control plane yourself and can inspect every flag and file. That is the best way to understand exactly what your cloud provider is doing on your behalf, and what they are not.
