---
foundation:
  topic: Kubernetes Security
  group: Secrets Management
  slug: overview
  image: assets/secrets-management.svg
  order: 70
  title: "Overview"
  description: "How Kubernetes stores sensitive data and why base64 is not encryption."
  publish: true
---

# Overview

A secret is any value that grants access when known: database passwords, API keys, TLS private keys, signing keys, OAuth client secrets, service credentials. Kubernetes has a built-in `Secret` object for these, and the single most important fact about it is this:

**Kubernetes Secrets are base64 encoded, not encrypted.**

Base64 is a transport encoding. Anyone who can read the object can decode the value with one command:

```bash
echo "cGFzc3dvcmQxMjM=" | base64 -d
```

```text
password123
```

Treat the encoding as packaging, not protection. The protection has to come from everything around the object.

## Where Secrets Leak

The threat surfaces to think about:

- **etcd:** by default, Secrets sit in etcd as plaintext (base64). Anyone with etcd access, or a copy of an etcd backup, has every secret in the cluster. Encryption at rest closes this; see the cluster hardening pages.
- **RBAC:** `get` or `list` on `secrets` in a namespace means reading every credential there. It is one of the most over-granted permissions in real clusters.
- **Environment variables vs volumes:** secrets injected as env vars leak through `/proc`, crash dumps, child processes, and debug endpoints that print the environment. Volume mounts are the safer default.
- **Logs and shell history:** `kubectl create secret --from-literal=...` puts the value in your shell history; applications that log their config print credentials into log pipelines.
- **Git:** manifests with inline secret data end up in repositories, where history never forgets.

## Three Approaches

This section covers three ways to manage secrets, in increasing order of separation from the cluster:

| Approach | Where the source of truth lives | Where the value ends up |
| --- | --- | --- |
| Native Secrets, hardened | The cluster | etcd + pod |
| External Secrets Operator | External manager (Vault, AWS, GCP, Azure) | Synced into etcd + pod |
| Secrets Store CSI | External manager | Mounted into the pod, etcd optional |

## How To Choose

- **Native Secrets** are fine for small clusters and low-value credentials, provided etcd encryption at rest is on and RBAC around `secrets` is tight.
- **External Secrets Operator** fits teams that already run a secrets manager and want rotation, audit, and one source of truth, while keeping the ergonomics of normal Kubernetes Secrets.
- **Secrets Store CSI** fits the strictest requirements, where secret material should reach only the pods that use it and ideally never rest in etcd at all.

Whichever path you take, two rules hold everywhere: never commit secret values to git, and never grant `get secrets` more widely than you can justify.

The next pages walk through each approach hands-on.
