---
foundation:
  topic: Kubernetes Security
  group: Cluster Hardening
  slug: etcd
  image: ../assets/cluster-hardening.svg
  order: 92
  title: "etcd"
  description: "Protecting the datastore that holds every cluster secret and object."
  publish: true
---

# etcd

etcd is the cluster's single source of truth. Every object the API server serves, including Deployments, ConfigMaps, and crucially every **Secret**, is a row in etcd. This has a blunt consequence: anyone who can read etcd can read every Secret in the cluster, and RBAC never enters the picture. RBAC guards the API server, not the database behind it.

## Prove Secrets Are Plaintext

Create a Secret through the normal path:

```bash
kubectl create namespace lab --dry-run=client -o yaml | kubectl apply -f -
kubectl -n lab create secret generic db-cred \
  --from-literal=password='S3cr3t-P@ss'
```

Now read it straight out of etcd, bypassing the API server entirely. etcd runs as a static pod on the control plane node with its client certs under `/etc/kubernetes/pki/etcd/`:

```bash
docker exec k8s-security-lab-control-plane sh -c '
  ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/secrets/lab/db-cred'
```

Expected output (abridged):

```text
/registry/secrets/lab/db-cred
k8s...v1Secret...db-cred...lab...
password
S3cr3t-P@ss
...
```

There it is: `S3cr3t-P@ss` in the clear. The Secret was never encrypted. Kubernetes only **base64-encodes** Secret values, which is encoding, not encryption. Anyone with etcd file access, an etcd backup, or a snapshot of the control plane disk has every credential in the cluster.

## Encryption At Rest

The fix is `EncryptionConfiguration`, which tells the API server to encrypt resources before writing them to etcd. It is a file on the control plane node referenced by `--encryption-provider-config`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <32-byte base64 key>
      - identity: {}
```

Key points about this file:

- The **first** provider is used to encrypt. `identity` (plaintext) is listed last so existing unencrypted Secrets stay readable during migration.
- After enabling it, force a rewrite so old Secrets get encrypted:
  ```bash
  kubectl get secrets --all-namespaces -o json | kubectl replace -f -
  ```
- The `aescbc` local-key provider stores the key **on the control plane node**. That protects against a stolen etcd backup, but not against someone who already has the node. For real protection, use a **KMS provider** that keeps the key in an external service (AWS KMS, GCP KMS, HashiCorp Vault) so the key never touches disk.

Managed clusters encrypt etcd at rest by default, and offer KMS-backed Secret encryption as a toggle. Enable it.

## TLS And Peer Authentication

etcd should trust no one it cannot verify:

- **Client-to-server TLS:** only the API server, holding a valid client cert, may talk to etcd. Check with:
  ```bash
  docker exec k8s-security-lab-control-plane \
    cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'client-cert-auth|cert-file|key-file'
  ```
  `--client-cert-auth=true` should be present.
- **Peer TLS:** in a multi-node etcd cluster, members authenticate to each other with peer certs (`--peer-client-cert-auth=true`).
- **Network isolation:** etcd's ports (2379/2380) should never be reachable from workloads or the internet. Nothing but the control plane needs them.

## Backups Are Secrets Too

An etcd snapshot is a complete, offline copy of every Secret in the cluster:

```bash
docker exec k8s-security-lab-control-plane sh -c '
  ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save /tmp/etcd-backup.db'
```

Treat that file with the same care as the cluster's most sensitive Secret, because it contains it:

1. Encrypt backups at rest and in transit.
2. Restrict who can read the backup bucket as tightly as production credentials.
3. Rotate and expire old snapshots, because a two-year-old backup still leaks today's long-lived Secrets.

## Clean Up

```bash
kubectl -n lab delete secret db-cred
docker exec k8s-security-lab-control-plane rm -f /tmp/etcd-backup.db
```

## Practical Guidance

1. Assume base64 Secret values are plaintext, because to an attacker they are.
2. Enable encryption at rest, and prefer a KMS provider over local `aescbc` keys so the key is not on the node.
3. After enabling encryption, rewrite existing Secrets so they are actually encrypted, because enabling it does not touch data already stored.
4. Lock etcd behind client-cert TLS and keep its ports off any network but the control plane's.
5. Guard etcd backups like the crown jewels they are: encrypted, access-controlled, and expired on a schedule.
6. On managed clusters, turn on KMS-based Secret encryption and confirm control plane disk encryption is on.
