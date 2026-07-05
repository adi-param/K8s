---
foundation:
  topic: Kubernetes Security
  group: Authorization
  slug: privilege-escalation
  image: ../assets/authorization.svg
  order: 34
  title: "Privilege Escalation"
  description: "Common RBAC misconfigurations that let a low privilege identity become cluster admin."
  publish: true
---

# Privilege Escalation

Privilege escalation happens when an identity can turn limited access into broader access.

In Kubernetes, escalation is often indirect. An identity may not be allowed to read Secrets directly, but it might be able to create a pod that mounts a powerful service account. It may not be cluster-admin, but it might be allowed to edit RBAC bindings.

## Permissions To Treat As Sensitive

Review these permissions carefully:

- **`secrets` read access:** can expose credentials, tokens, and application secrets.
- **`pods/exec`:** can open a shell inside workloads.
- **`pods/log`:** can expose tokens or sensitive application output.
- **`create pods`:** can run code, mount volumes, or use service accounts.
- **`update deployments` or other workload controllers:** can change running application behavior.
- **`impersonate`:** can make requests as another user or group.
- **`bind` and `escalate`:** can grant permissions beyond the caller's current access.
- **RBAC write access:** can create or modify Roles, ClusterRoles, and bindings.
- **admission policy write access:** can weaken controls that block unsafe workloads.

These are not always wrong, but they should be intentional.

## Pod Creation Can Be Powerful

If an identity can create pods in a namespace, ask what service accounts exist in that namespace.

Check service accounts:

```bash
kubectl get serviceaccounts --namespace=lab
```

Check whether an identity can create pods:

```bash
kubectl auth can-i create pods \
  --as system:serviceaccount:lab:some-service-account \
  --namespace=lab
```

If a powerful service account exists and pods can use it, pod creation may become privilege escalation.

## RBAC Write Access Is High Risk

An identity that can create RoleBindings may be able to grant permissions to itself or another identity.

Check RBAC write permissions:

```bash
kubectl auth can-i create rolebindings --namespace=lab
kubectl auth can-i create clusterrolebindings
kubectl auth can-i update clusterroles
```

Kubernetes includes escalation checks for RBAC changes, but you should still treat RBAC write access as administrative.

## Impersonation

Impersonation lets one identity make requests as another identity:

```bash
kubectl auth can-i list pods --as alice
```

This is useful for administrators testing access, but dangerous if granted too broadly. An identity with broad impersonation can bypass the intended separation between users, groups, and service accounts.

## Safer Patterns

Reduce escalation paths with these habits:

1. Use dedicated service accounts for workloads.
2. Disable service account token mounting when pods do not need API access.
3. Keep powerful service accounts out of namespaces where many people can create pods.
4. Limit who can create pods, exec into pods, and read logs.
5. Avoid granting write access to RBAC resources to application teams.
6. Use admission controls to block privileged pods, hostPath mounts, host namespaces, and unsafe capabilities.
7. Review ClusterRoleBindings regularly.

## Review Commands

Useful checks:

```bash
kubectl get rolebindings,clusterrolebindings --all-namespaces
kubectl auth can-i --list --namespace=lab
kubectl auth can-i create pods --namespace=lab
kubectl auth can-i get secrets --namespace=lab
kubectl auth can-i create clusterrolebindings
```

RBAC is not only about what an identity can do right now. It is also about whether that identity can create a path to more power.
