---
foundation:
  topic: Kubernetes Security
  group: Authorization
  slug: roles
  image: ../assets/authorization.svg
  order: 32
  title: "Roles"
  description: "Namespaced permission sets and how to scope access to a single namespace."
  publish: true
---

# Roles

Roles define permissions inside one namespace.

Use a Role when an application, team, or automation process only needs access to resources in a specific namespace. This is the safest default for most application workloads.

## Role Example

This Role allows reading ConfigMaps in the `lab` namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: lab
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
```

Apply it:

```bash
kubectl apply -f - <<'YAML'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: lab
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
YAML
```

## Bind The Role

Create a service account:

```bash
kubectl create serviceaccount configmap-reader-sa --namespace=lab
```

Create a RoleBinding:

```bash
kubectl create rolebinding configmap-reader-binding \
  --role=configmap-reader \
  --serviceaccount=lab:configmap-reader-sa \
  --namespace=lab
```

Now check access:

```bash
kubectl auth can-i list configmaps \
  --as system:serviceaccount:lab:configmap-reader-sa \
  --namespace=lab
```

Expected output:

```text
yes
```

Check access in another namespace:

```bash
kubectl auth can-i list configmaps \
  --as system:serviceaccount:lab:configmap-reader-sa \
  --namespace=default
```

Expected output:

```text
no
```

That is the point of a Role: the permission stays inside the namespace where it is bound.

## Resource Names

Roles can limit access to specific object names:

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get"]
```

This can be useful, but it has limits. For example, `list` and `watch` usually do not work cleanly with `resourceNames` because those verbs operate over collections.

## Clean Up

```bash
kubectl delete rolebinding configmap-reader-binding --namespace=lab
kubectl delete role configmap-reader --namespace=lab
kubectl delete serviceaccount configmap-reader-sa --namespace=lab
```

## Practical Guidance

Use Roles when:

- the workload only needs one namespace
- a team owns one namespace
- an automation job operates on one application environment
- you want a clear permission boundary

Avoid using Roles to grant access to sensitive resources by habit. Reading Secrets, creating pods, using `pods/exec`, and updating workloads can all create paths to higher privilege.
