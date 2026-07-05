---
foundation:
  topic: Kubernetes Security
  group: Identity
  slug: overview
  image: assets/identity.svg
  order: 20
  title: "Overview"
  description: "How Kubernetes decides who you are: the difference between authentication and authorization, and the identity types a cluster understands."
  publish: true
---

# Overview

Identity answers one question: **who is making this request to the Kubernetes API server?**

Kubernetes separates that from authorization. Authentication identifies the caller. Authorization decides what that caller is allowed to do.

```text
request -> authentication -> authorization -> admission -> stored object
```

If authentication fails, the request is rejected before permissions are checked. If authentication succeeds, Kubernetes attaches a username and groups to the request, then passes that identity to the authorization layer.

## Identity Types

Kubernetes commonly sees two broad identity types:

- **Human identities**, such as engineers, operators, CI/CD users, or break-glass administrators.
- **Workload identities**, usually service accounts used by pods inside the cluster.

Human login is usually handled outside Kubernetes by a cloud provider, identity provider, or certificate process. Workload identity is native to Kubernetes through ServiceAccount objects and projected tokens.

## What Kubernetes Stores

Kubernetes does not normally store user accounts as API objects. You will not find a built-in `User` resource like you find `Pod`, `Role`, or `ServiceAccount`.

Instead, the API server trusts configured authentication methods, such as:

- x509 client certificates
- OpenID Connect tokens
- webhook token authentication
- service account tokens
- cloud-provider integrations

After authentication, the API server works with a username and groups. RBAC rules can then refer to those identities.

## Why Identity Matters

Bad identity design creates broad blast radius. Common problems include:

- long-lived credentials that are hard to revoke
- shared admin kubeconfigs
- service accounts with unnecessary permissions
- pods automatically receiving API tokens they do not need
- CI/CD systems using cluster-admin credentials for routine deployments

A safer baseline is simple:

1. Use external identity for humans where possible.
2. Avoid shared user credentials.
3. Give workloads dedicated service accounts.
4. Bind permissions to groups or service accounts, not individual ad hoc identities.
5. Prefer short-lived tokens over long-lived secrets.
6. Regularly check what identities can do with `kubectl auth can-i`.

The next pages cover three identity mechanisms you will see often: client certificates, OIDC, and service accounts.
