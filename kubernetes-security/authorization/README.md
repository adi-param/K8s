---
foundation:
  topic: Kubernetes Security
  group: Authorization
  slug: overview
  image: assets/authorization.svg
  order: 30
  title: "Overview"
  description: "How Kubernetes decides what an authenticated identity is allowed to do, and the authorization modes the API server chains together."
  publish: true
---

# Overview

Authorization answers one question: **what is this authenticated identity allowed to do?**

Authentication gives Kubernetes a username and groups. Authorization checks that identity against policy before the request can continue.

```text
request -> authentication -> authorization -> admission -> stored object
```

Most clusters use RBAC as the main authorization system.

## What Authorization Checks

Kubernetes authorization is action based. A request is evaluated using:

- **subject:** the user, group, or service account making the request
- **verb:** the action, such as `get`, `list`, `create`, `update`, or `delete`
- **resource:** the API resource, such as `pods`, `secrets`, or `deployments`
- **namespace:** the namespace for namespaced resources
- **resource name:** an optional specific object name

For example:

```text
Can system:serviceaccount:lab:app-reader get pods in namespace lab?
```

## Common Building Blocks

RBAC uses four main objects:

- **Role:** permissions inside one namespace.
- **RoleBinding:** grants a Role to users, groups, or service accounts.
- **ClusterRole:** permissions that can apply cluster-wide or be reused in namespaces.
- **ClusterRoleBinding:** grants a ClusterRole across the whole cluster.

The safe default is to start with namespaced `Role` and `RoleBinding` objects. Use cluster-wide grants only when the access really must cross namespaces or target cluster-scoped resources.

## Check Access

Use `kubectl auth can-i` before and after creating permissions:

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments
kubectl auth can-i get secrets
```

Check as a service account:

```bash
kubectl auth can-i get pods \
  --as system:serviceaccount:lab:app-reader \
  --namespace lab
```

This command is one of the simplest ways to verify RBAC behavior while learning.

## Baseline Rules

Start with these habits:

1. Grant the smallest set of verbs and resources that still works.
2. Prefer namespace-scoped Roles over ClusterRoles for application access.
3. Bind permissions to groups or service accounts, not shared admin users.
4. Avoid broad verbs like `*` unless there is a strong reason.
5. Treat access to `secrets`, `pods/exec`, `pods/log`, and workload creation as sensitive.
6. Review ClusterRoleBindings carefully because they apply across the cluster.

Authorization is where identity becomes real power. The next pages show how RBAC, Roles, ClusterRoles, and common escalation paths work.
