---
foundation:
  topic: Kubernetes Security
  group: Network Security
  slug: network-policies
  image: ../assets/network-security.svg
  order: 61
  title: "Network Policies"
  description: "Declaratively allowing and denying pod to pod and pod to external traffic."
  publish: true
---

# Network Policies

A `NetworkPolicy` selects pods by label and lists the traffic they may receive and send. There are no deny rules and no ordering: you deny by selecting, and allow by listing.

## The Policy Model

Every policy has three parts:

- **`podSelector`:** which pods this policy applies to. An empty selector `{}` means every pod in the namespace.
- **`policyTypes`:** which directions this policy governs: `Ingress`, `Egress`, or both.
- **`ingress` / `egress` rules:** the allowed peers (`podSelector`, `namespaceSelector`, `ipBlock`) and ports.

The key mental model: **selection flips the default.** A pod selected by no ingress policy accepts all traffic. Once any ingress policy selects it, only traffic matched by some rule is allowed. The same holds independently for egress. Policies are additive: the allowed set is the union of every policy that selects the pod.

## A Lab That Actually Enforces

kind's default CNI (`kindnet`) stores policies but does not enforce them, so this lab needs a cluster with an enforcing CNI. Create one with Calico:

```bash
cat <<'EOF' | kind create cluster --name k8s-netpol-lab --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
kubectl create namespace lab
```

If you run this against the regular `k8s-security-lab` cluster instead, every step will appear to work, including the deny, because nothing enforces the policy. That false confidence is exactly why you verify CNI enforcement with a real test.

## Watch Default-Deny Take Effect

Start a web server and a client:

```bash
kubectl run web --image=nginx:1.27 -n lab --labels=app=web
kubectl run client --image=busybox:1.36 -n lab --labels=app=client -- sleep 3600
kubectl exec -n lab client -- wget -qO- --timeout=2 http://web-pod-ip
```

Get the pod IP with `kubectl get pod web -n lab -o wide`. Expected output: the nginx welcome page, the flat network at work.

Now deny all ingress in the namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: lab
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

```bash
kubectl apply -f default-deny-ingress.yaml
kubectl exec -n lab client -- wget -qO- --timeout=2 http://web-pod-ip
```

Expected output:

```text
wget: download timed out
command terminated with exit code 1
```

## Allow One Flow Back

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-web
  namespace: lab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: client
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f allow-client-to-web.yaml
kubectl exec -n lab client -- wget -qO- --timeout=2 http://web-pod-ip | head -4
```

Expected output:

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

The client reaches `web` again; every other pod in the namespace still cannot. This pair, one default-deny and one named allow per real flow, is the whole pattern.

## Selecting Peers Across Namespaces

Rules can match peers three ways, and combining them matters:

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
        podSelector:
          matchLabels:
            app: prometheus
```

As one list item, `namespaceSelector` and `podSelector` are ANDed: prometheus pods in the monitoring namespace. As two separate list items (each with its own `-`), they are ORed: all pods in monitoring, plus prometheus pods in this namespace. This indentation difference is the most common NetworkPolicy bug.

`ipBlock` matches CIDR ranges and is mainly for cluster-external addresses; pod IPs are ephemeral and should be matched by label, never by CIDR.

## Clean Up

```bash
kind delete cluster --name k8s-netpol-lab
```

## Practical Guidance

1. Verify your CNI enforces NetworkPolicy with a live deny test before trusting any policy.
2. Apply `default-deny-ingress` to every application namespace as part of namespace provisioning.
3. Write one policy per real traffic flow and name it after the flow: `allow-client-to-web`.
4. Select by labels, not IPs; reserve `ipBlock` for external CIDRs.
5. Watch the AND/OR indentation with `namespaceSelector` plus `podSelector`; it inverts the meaning.
6. Remember policies are namespaced: a namespace with no policies is still wide open.
