---
foundation:
  topic: Kubernetes Security
  group: Identity
  slug: service-accounts
  image: ../assets/identity.svg
  order: 23
  title: "Service Accounts"
  description: "How workloads authenticate to the API server using service account tokens."
  publish: true
---

# Service Accounts

Service accounts are Kubernetes identities for workloads.

When a pod needs to call the Kubernetes API, it should use a service account with only the permissions that workload needs. This is different from a human user logging in with `kubectl`.

## Default Behavior

Every namespace has a `default` service account:

```bash
kubectl get serviceaccounts
```

If you create a pod without specifying a service account, Kubernetes uses the namespace's `default` service account.

That is convenient, but it is not a good security habit. Important workloads should have explicit service accounts so their permissions are easy to reason about.

## Create a Dedicated Service Account

Create one in the `lab` namespace:

```bash
kubectl create serviceaccount app-reader
```

Create a pod that uses it:

```bash
kubectl run sa-demo \
  --image=busybox:1.36 \
  --restart=Never \
  --serviceaccount=app-reader \
  --command -- sleep 3600
```

Check which service account the pod uses:

```bash
kubectl get pod sa-demo -o jsonpath='{.spec.serviceAccountName}'; echo
```

Expected output:

```text
app-reader
```

## Service Account Tokens

Modern Kubernetes uses short-lived projected service account tokens. Inside a pod, the token is usually mounted here:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

That token lets the workload authenticate to the API server as:

```text
system:serviceaccount:<namespace>:<service-account-name>
```

For the example above:

```text
system:serviceaccount:lab:app-reader
```

RBAC rules can grant permissions to that service account.

## Avoid Unneeded Tokens

Many pods do not need to call the Kubernetes API. For those pods, disable automatic token mounting:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-api-token
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

You can also set this on the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
automountServiceAccountToken: false
```

This reduces the value of a compromised pod because there may be no Kubernetes API token to steal.

## Check Permissions

Use `kubectl auth can-i` to ask what a service account can do:

```bash
kubectl auth can-i get pods \
  --as system:serviceaccount:lab:app-reader
```

Check a more sensitive action:

```bash
kubectl auth can-i create clusterroles \
  --as system:serviceaccount:lab:app-reader
```

For a newly created service account with no RBAC bindings, these should usually return `no`.

## Clean Up

Remove the test pod and service account:

```bash
kubectl delete pod sa-demo
kubectl delete serviceaccount app-reader
```

## Practical Guidance

Use these rules:

1. Create a dedicated service account for each meaningful workload.
2. Do not rely on the namespace `default` service account for important apps.
3. Disable token mounting when a pod does not need Kubernetes API access.
4. Grant permissions with RBAC only after you know what the workload needs.
5. Treat service account tokens as sensitive credentials.

Service accounts are the bridge between identity and authorization for workloads. The next section, Authorization, covers how to grant permissions safely.
