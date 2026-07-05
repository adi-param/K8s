---
foundation:
  topic: Kubernetes Security
  group: Secrets Management
  slug: secrets-store-csi
  image: ../assets/secrets-management.svg
  order: 73
  title: "Secrets Store CSI"
  description: "Mounting secrets from an external provider straight into pods with the CSI driver."
  publish: true
---

# Secrets Store CSI

The Secrets Store CSI Driver takes the external-manager idea one step further than the External Secrets Operator: instead of syncing values into Kubernetes Secrets, it mounts them from the external store **directly into the pod's filesystem** at pod start. In its default mode, the secret value never becomes a Kubernetes Secret and never rests in etcd.

## Architecture

Two components run on every node:

- **The CSI driver** (a DaemonSet): implements the volume mount and hands requests to a provider.
- **A provider plugin** per external manager: AWS, Azure, GCP, or Vault. The provider authenticates (ideally with the pod's own workload identity), fetches the values, and the driver writes them into a tmpfs volume inside the pod.

The flow at pod start:

```text
pod scheduled -> kubelet mounts CSI volume -> provider fetches from store
             -> files appear under the mountPath -> containers start
```

If the store is unreachable or access is denied, the pod fails to start. That is a feature: no pod runs with missing credentials.

## SecretProviderClass

A `SecretProviderClass` describes what to fetch. This example targets AWS Secrets Manager:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: db-creds
  namespace: lab
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db/password"
        objectType: "secretsmanager"
```

The pod references it as a CSI volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: lab
spec:
  serviceAccountName: app            # carries the workload identity (e.g. IRSA)
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: creds
          mountPath: /mnt/secrets
          readOnly: true
  volumes:
    - name: creds
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: db-creds
```

Inside the container, the secret is a file:

```bash
kubectl exec -n lab app -- cat /mnt/secrets/prod_db_password
```

There is no Secret object to leak, no etcd record, and RBAC on `secrets` is not in the picture. Access control lives entirely in the external store's policy for that pod's identity.

A full hands-on run needs a real provider (a cloud account or a Vault install), so treat the manifests above as the flow to reproduce in your environment; the driver and each provider ship helm charts (`secrets-store-csi-driver` plus e.g. `secrets-store-csi-driver-provider-aws`).

## Optional Sync To Kubernetes Secrets

Some things can only consume Kubernetes Secrets, such as env vars and ingress TLS references. The driver can optionally materialize one alongside the mount using `secretObjects` in the `SecretProviderClass`. Note the caveats: the sync only happens while at least one pod mounts the volume, and once synced, the value is back in etcd with all the implications from the earlier pages.

## CSI Driver Or External Secrets Operator

| | Secrets Store CSI | External Secrets Operator |
| --- | --- | --- |
| Delivery | Mounted at pod start | Synced continuously into Secrets |
| In etcd | No (unless sync enabled) | Yes |
| Consumable as env vars | Only via sync | Yes |
| Rotation | Optional rotation reconciler updates mounted files | Refresh interval updates the Secret |
| Store outage | New pods fail to start | Pods unaffected; Secret goes stale |
| Runs as | DaemonSet on every node + provider | Single controller deployment |

A reasonable rule: ESO for general use and ergonomics, CSI for workloads whose secrets must not rest in etcd, and both only after native Secret hygiene (RBAC, encryption at rest) is in place, since most clusters end up with some native Secrets regardless.

## Clean Up

```bash
kubectl delete pod app -n lab
kubectl delete secretproviderclass db-creds -n lab
```

## Practical Guidance

1. Authenticate providers with the pod's workload identity (IRSA, Workload Identity, Vault Kubernetes auth) so each pod can read exactly its own secrets and nothing else.
2. Prefer the pure-mount mode; enable `secretObjects` sync only for consumers that genuinely cannot read files.
3. Enable the rotation reconciler if your store rotates credentials, and make applications re-read the mounted files.
4. Plan for store availability: the external manager is now on the pod-startup critical path.
5. Keep the driver and provider plugins patched; they run on every node and handle every secret in transit.
