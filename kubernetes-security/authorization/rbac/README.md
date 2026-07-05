---
foundation:
  topic: Kubernetes Security
  group: Authorization
  slug: rbac
  image: ../assets/authorization.svg
  order: 31
  title: "RBAC"
  description: "Role based access control: the primary way to grant least privilege access in a cluster."
  publish: true
---

# RBAC

RBAC, or Role-Based Access Control, is the primary way Kubernetes grants permissions.

RBAC does not describe job titles. It describes allowed API actions. A Role might allow reading pods, creating deployments, or updating ConfigMaps. A binding attaches those permissions to an identity.

## The RBAC Model

RBAC has two permission objects and two binding objects:

| Object | Scope | Purpose |
| --- | --- | --- |
| `Role` | Namespace | Defines permissions in one namespace |
| `RoleBinding` | Namespace | Grants a Role or ClusterRole inside one namespace |
| `ClusterRole` | Cluster | Defines cluster-wide permissions or reusable permission sets |
| `ClusterRoleBinding` | Cluster | Grants a ClusterRole across the whole cluster |

The split matters. A `ClusterRole` is only cluster-wide when it is granted with a `ClusterRoleBinding`. If a `ClusterRole` is granted with a `RoleBinding`, the access is limited to that RoleBinding's namespace.

## Verbs And Resources

RBAC rules are built from verbs and resources:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Common verbs include:

- `get`: read one object
- `list`: list objects
- `watch`: stream changes
- `create`: create new objects
- `update`: replace existing objects
- `patch`: change part of an object
- `delete`: remove objects

Some permissions are more powerful than they look. For example, `create pods` can be dangerous if the identity can create privileged pods or mount sensitive service accounts. `get secrets` can expose credentials.

## Create A Read-Only Role

Create a Role that can read pods in the `lab` namespace:

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=lab
```

Create a service account:

```bash
kubectl create serviceaccount pod-reader-sa --namespace=lab
```

Bind the Role to the service account:

```bash
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=lab:pod-reader-sa \
  --namespace=lab
```

Check the result:

```bash
kubectl auth can-i list pods \
  --as system:serviceaccount:lab:pod-reader-sa \
  --namespace=lab
```

Expected output:

```text
yes
```

Check a permission that was not granted:

```bash
kubectl auth can-i delete pods \
  --as system:serviceaccount:lab:pod-reader-sa \
  --namespace=lab
```

Expected output:

```text
no
```

## Clean Up

```bash
kubectl delete rolebinding pod-reader-binding --namespace=lab
kubectl delete role pod-reader --namespace=lab
kubectl delete serviceaccount pod-reader-sa --namespace=lab
```

## Practical Guidance

Use RBAC like a narrow allow list:

1. Grant only the verbs needed.
2. Grant only the resources needed.
3. Prefer namespaced access for applications.
4. Avoid `*` in verbs, resources, or API groups.
5. Test permissions with `kubectl auth can-i`.
6. Review access to secrets, token creation, exec, impersonation, and workload creation especially carefully.
