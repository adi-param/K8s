---
foundation:
  topic: Kubernetes Security
  group: Authorization
  slug: cluster-roles
  image: ../assets/authorization.svg
  order: 33
  title: "ClusterRoles"
  description: "Cluster wide permission sets for resources that live outside any namespace."
  publish: true
---

# ClusterRoles

ClusterRoles define permissions that are not tied to one namespace.

They are needed for cluster-scoped resources, such as nodes, namespaces, persistent volumes, and custom resource definitions. They can also define reusable permission sets that are later bound inside a namespace with a RoleBinding.

## ClusterRole vs Role

Use a Role for namespaced access:

```text
Role + RoleBinding -> access inside one namespace
```

Use a ClusterRole for cluster-scoped resources or reusable rules:

```text
ClusterRole + RoleBinding -> access inside one namespace
ClusterRole + ClusterRoleBinding -> access across the cluster
```

That distinction is important. A ClusterRole is not automatically granted everywhere. The binding decides where it applies.

## Reuse A ClusterRole In One Namespace

Kubernetes includes built-in ClusterRoles such as `view`, `edit`, and `admin`.

Create a service account:

```bash
kubectl create serviceaccount namespace-viewer --namespace=lab
```

Bind the built-in `view` ClusterRole only in the `lab` namespace:

```bash
kubectl create rolebinding namespace-viewer-binding \
  --clusterrole=view \
  --serviceaccount=lab:namespace-viewer \
  --namespace=lab
```

Check access in `lab`:

```bash
kubectl auth can-i list pods \
  --as system:serviceaccount:lab:namespace-viewer \
  --namespace=lab
```

Expected output:

```text
yes
```

Check access in `default`:

```bash
kubectl auth can-i list pods \
  --as system:serviceaccount:lab:namespace-viewer \
  --namespace=default
```

Expected output:

```text
no
```

## Cluster-Wide Binding

A ClusterRoleBinding applies across the cluster:

```bash
kubectl create clusterrolebinding namespace-viewer-cluster-wide \
  --clusterrole=view \
  --serviceaccount=lab:namespace-viewer
```

Now that service account can view resources in every namespace where the `view` role grants access.

Delete the cluster-wide binding after testing:

```bash
kubectl delete clusterrolebinding namespace-viewer-cluster-wide
```

## Clean Up

```bash
kubectl delete rolebinding namespace-viewer-binding --namespace=lab
kubectl delete serviceaccount namespace-viewer --namespace=lab
```

## Practical Guidance

Use ClusterRoles carefully:

1. Use ClusterRoleBindings only when access must be cluster-wide.
2. Prefer RoleBindings when reusing a ClusterRole inside one namespace.
3. Review built-in roles before using them; `view`, `edit`, and `admin` are broad.
4. Treat access to nodes, namespaces, CRDs, admission resources, and RBAC resources as sensitive.
5. Avoid binding service accounts to `cluster-admin` except for tightly controlled platform components.
