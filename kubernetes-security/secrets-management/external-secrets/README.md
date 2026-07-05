---
foundation:
  topic: Kubernetes Security
  group: Secrets Management
  slug: external-secrets
  image: ../assets/secrets-management.svg
  order: 72
  title: "External Secrets"
  description: "Syncing secrets from an external store into the cluster with the External Secrets Operator."
  publish: true
---

# External Secrets

The External Secrets Operator (ESO) makes an external secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault) the source of truth, and syncs values into ordinary Kubernetes Secrets.

Applications keep consuming plain Secrets; nothing in the pod spec changes. What changes is where the value lives and who manages its lifecycle: rotation, audit, and access control happen in the external manager, and git only ever contains *references* to secrets, never values.

ESO uses two main resources:

- **SecretStore / ClusterSecretStore:** how to reach and authenticate to the external manager (namespaced / cluster-wide).
- **ExternalSecret:** which keys to fetch and what Kubernetes Secret to materialize from them.

## Install

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

Verify:

```bash
kubectl get pods -n external-secrets
```

Expected output:

```text
NAME                                                READY   STATUS    RESTARTS   AGE
external-secrets-...                                1/1     Running   0          60s
external-secrets-cert-controller-...                1/1     Running   0          60s
external-secrets-webhook-...                        1/1     Running   0          60s
```

## A Local Lab With The Fake Provider

Real providers need cloud credentials. For learning on the kind cluster, ESO ships a `fake` provider that serves values you define inline. The resource flow is identical to production:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: fake-store
  namespace: lab
spec:
  provider:
    fake:
      data:
        - key: /prod/db/password
          value: "S3cretFromStore"
```

Now declare which keys to sync:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-password
  namespace: lab
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: fake-store
    kind: SecretStore
  target:
    name: db-password
  data:
    - secretKey: password
      remoteRef:
        key: /prod/db/password
```

Apply both and watch the Secret appear:

```bash
kubectl apply -f fake-store.yaml -f external-secret.yaml
kubectl get externalsecret db-password -n lab
```

Expected output:

```text
NAME          STORETYPE     STORE        REFRESH INTERVAL   STATUS         READY
db-password   SecretStore   fake-store   1h                 SecretSynced   True
```

The materialized Secret is a completely normal one:

```bash
kubectl get secret db-password -n lab \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

```text
S3cretFromStore
```

In production the only change is the `provider` block, for example `aws.service: SecretsManager` with IRSA authentication, or `vault` with a Kubernetes auth role.

## Rotation

`refreshInterval` controls how often ESO re-reads the external store. Rotate the value in the manager and the Kubernetes Secret follows on the next refresh; pods consuming it as a volume mount pick up the new file without a restart. This is the loop that makes short-lived credentials practical.

## The Trade-Off

ESO synchronizes secrets *into* the cluster. The value still lands in etcd, and RBAC on the materialized Secret still matters. Everything from the previous page continues to apply, including encryption at rest. What you gain is lifecycle: one audited source of truth, rotation, and no secret values in git. If the requirement is that secrets never rest in etcd at all, that is the Secrets Store CSI driver on the next page.

Guard the stores themselves: whoever can edit a `SecretStore` or `ExternalSecret` can potentially redirect or exfiltrate secrets, so treat those RBAC grants like secret access.

## Clean Up

```bash
kubectl delete externalsecret db-password -n lab
kubectl delete secretstore fake-store -n lab
helm uninstall external-secrets -n external-secrets
kubectl delete namespace external-secrets
```

## Practical Guidance

1. Make the external manager the single source of truth; git carries only `ExternalSecret` references.
2. Authenticate stores with workload identity (IRSA, Workload Identity, Vault Kubernetes auth), never with a long-lived cloud key stored as, ironically, a Kubernetes Secret.
3. Set `refreshInterval` to match your rotation policy, and mount synced secrets as volumes so rotation propagates without restarts.
4. Keep etcd encryption at rest on; synced secrets are still cluster secrets.
5. Restrict who can create or edit `SecretStore`, `ClusterSecretStore`, and `ExternalSecret` objects; they are part of the credential path.
