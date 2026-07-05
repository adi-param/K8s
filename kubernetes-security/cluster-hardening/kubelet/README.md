---
foundation:
  topic: Kubernetes Security
  group: Cluster Hardening
  slug: kubelet
  image: ../assets/cluster-hardening.svg
  order: 93
  title: "Kubelet"
  description: "Locking down the node agent so a compromised node cannot be used to pivot."
  publish: true
---

# Kubelet

The kubelet runs on every node and does the actual work: starting containers, streaming logs, and running `exec` sessions. It exposes an HTTP API on port 10250 to do this. That API is a second entrance to the cluster, less watched than the API server, and just as powerful on the node it controls.

If the kubelet API accepts unauthenticated requests, an attacker who reaches port 10250 can list pods, read their logs, and exec into them (reading Secrets from mounted volumes and environment variables) **without ever authenticating to the API server or touching RBAC**.

## Read The Kubelet Config

On kubeadm and kind, the kubelet is configured by a file on the node:

```bash
docker exec k8s-security-lab-control-plane \
  cat /var/lib/kubelet/config.yaml
```

Three sections decide the kubelet's exposure.

## Anonymous Access

```bash
docker exec k8s-security-lab-control-plane \
  grep -A3 'authentication:' /var/lib/kubelet/config.yaml
```

Expected output:

```text
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
```

- `anonymous.enabled: false` rejects unauthenticated requests. This must be false. The historically dangerous setting is `true`, which let anyone reaching the port run kubelet API calls.
- `webhook.enabled: true` lets the kubelet verify bearer tokens against the API server, so clients must present a real identity.

## Authorization Mode

```bash
docker exec k8s-security-lab-control-plane \
  grep -A2 'authorization:' /var/lib/kubelet/config.yaml
```

Expected output:

```text
authorization:
  mode: Webhook
```

`Webhook` means the kubelet asks the API server "is this caller allowed to do this?" for every request, using the same RBAC rules as everything else. The unsafe alternative is `AlwaysAllow`, which authenticates the caller and then permits any action, precisely what you do not want on a node agent.

## The Read-Only Port

Older kubelets served an **unauthenticated read-only API** on port 10255. It required no auth and happily listed pods and their specs, including mounted Secret names and environment variables. It must be disabled:

```bash
docker exec k8s-security-lab-control-plane \
  grep -i readOnlyPort /var/lib/kubelet/config.yaml
```

You want `readOnlyPort: 0` (the modern default). If the field is absent on an older cluster, set `--read-only-port=0` explicitly. There is no legitimate use for an unauthenticated cluster API in production.

## Serving Certificates

The kubelet serves TLS on 10250. By default it uses a self-signed certificate, which means the API server cannot verify it is talking to the real kubelet (relevant for `kubectl logs` and `exec`, which the API server proxies to the kubelet). Enabling `serverTLSBootstrap: true` makes the kubelet request a proper serving cert from the cluster CA, which an approver (or the built-in signer) then issues. This closes a man-in-the-middle gap on the control-plane-to-kubelet path.

## NodeRestriction Ties It Together

Kubelet hardening pairs with two API server settings from the API Server page:

- The **`Node` authorizer** limits each kubelet to reading and modifying only the objects tied to its own node and its own pods.
- The **`NodeRestriction`** admission plugin stops a kubelet from editing other nodes or labeling itself to attract sensitive workloads.

Together they mean a single compromised node cannot be used to pivot into controlling the whole cluster. Confirm both are on:

```bash
docker exec k8s-security-lab-control-plane sh -c '
  grep -E "authorization-mode|enable-admission-plugins" \
    /etc/kubernetes/manifests/kube-apiserver.yaml'
```

You should see `Node` in the authorization mode and `NodeRestriction` in the admission plugins.

## Clean Up

Nothing to clean up. This lab only read configuration.

## Practical Guidance

1. Confirm `authentication.anonymous.enabled: false` on every node; anonymous kubelet access is a direct path to pod Secrets.
2. Confirm `authorization.mode: Webhook`, never `AlwaysAllow`.
3. Set `readOnlyPort: 0` and verify nothing depends on 10255.
4. Enable `serverTLSBootstrap` so kubelet serving certs are signed by the cluster CA rather than self-signed.
5. Keep the `Node` authorizer and `NodeRestriction` admission plugin enabled so one bad node cannot escalate to the cluster.
6. Firewall ports 10250 and 10255 so only the control plane can reach the kubelet API; on managed clusters, keep node pools off public subnets.
