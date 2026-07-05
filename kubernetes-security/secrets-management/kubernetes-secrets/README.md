---
foundation:
  topic: Kubernetes Security
  group: Secrets Management
  slug: kubernetes-secrets
  image: ../assets/secrets-management.svg
  order: 71
  title: "Kubernetes Secrets"
  description: "The built in Secret object, encryption at rest, and its limitations."
  publish: true
---

# Kubernetes Secrets

The `Secret` object is Kubernetes' built-in container for sensitive data. It is convenient and well integrated, but easy to overtrust. This page shows what it actually protects and how to harden its use.

## Base64 Is Encoding, Not Encryption

Create a secret:

```bash
kubectl create secret generic db-creds \
  --from-literal=username=appuser \
  --from-literal=password=S3cretP@ss \
  --namespace=lab
```

Now read it back:

```bash
kubectl get secret db-creds -n lab \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Expected output:

```text
S3cretP@ss
```

No key, no decryption, no privilege beyond `get secrets`. The value was never protected, only encoded. Two consequences follow:

1. Anyone with read access to the Secret has the credential.
2. Anyone with access to etcd, or an etcd backup, has every credential, unless encryption at rest is enabled.

Encryption at rest is an API server setting covered in the cluster hardening pages under etcd. Without it, treat etcd and its backups as a plaintext credential store.

## Volume Mounts Over Environment Variables

Pods consume Secrets two ways, and they are not equally safe.

**Environment variables** spread the value around: it appears in `/proc/<pid>/environ`, is inherited by every child process, shows up in crash dumps, and is printed by any framework debug page or logger that dumps the environment. Anyone who can `kubectl exec` into the pod can read it instantly:

```bash
kubectl exec -n lab mypod -- env | grep PASSWORD
```

**Volume mounts** keep the value in a file with normal file permissions, visible only to processes that deliberately read that path:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol
  namespace: lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: creds
          mountPath: /etc/creds
          readOnly: true
  volumes:
    - name: creds
      secret:
        secretName: db-creds
```

```bash
kubectl apply -f secret-vol.yaml
kubectl exec -n lab secret-vol -- cat /etc/creds/password; echo
```

Expected output:

```text
S3cretP@ss
```

Mounted secrets have one more advantage: when the Secret is updated, the kubelet refreshes the mounted files automatically (after a short delay). Environment variables are fixed at container start.

## RBAC Around Secrets

`get` and `list` on `secrets` deserve the same scrutiny as cluster-admin. Check who has it:

```bash
kubectl auth can-i get secrets --namespace=lab
kubectl auth can-i list secrets --as system:serviceaccount:lab:default --namespace=lab
```

Expected output for the default service account:

```text
no
```

Keep it that way. Grant secret access per name where possible:

```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-creds"]
    verbs: ["get"]
```

Note that `list` cannot be restricted by `resourceNames`. An identity with `list secrets` reads everything in the namespace regardless.

## Immutable Secrets

Marking a Secret immutable prevents accidental or malicious edits and lets the kubelet stop watching it:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds-v2
  namespace: lab
immutable: true
stringData:
  password: S3cretP@ss
```

Rotation then becomes creating `db-creds-v3` and updating the pod spec: a deliberate, auditable change.

## Keep Secrets Out Of Git

A Secret manifest with real data in it must never be committed. History rewrites are painful and forks never forget. If you want secrets in git for GitOps workflows, encrypt them first: **Sealed Secrets** encrypts to a controller-held key, and **SOPS** encrypts values with KMS/age keys while leaving the structure diffable. Both make the committed file useless without the decryption key.

## Clean Up

```bash
kubectl delete pod secret-vol -n lab
kubectl delete secret db-creds db-creds-v2 -n lab
```

## Practical Guidance

1. Enable etcd encryption at rest before storing anything real in Secrets; until then, base64 in etcd is plaintext.
2. Mount secrets as volumes, not environment variables.
3. Audit every grant of `get`/`list`/`watch` on `secrets`; scope to `resourceNames` where you can.
4. Mark stable credentials `immutable: true` and rotate by replacement.
5. Never commit secret values; use Sealed Secrets or SOPS if GitOps needs them in a repository.
6. Prefer short-lived, externally managed credentials over long-lived Secrets; the next two pages show how.
