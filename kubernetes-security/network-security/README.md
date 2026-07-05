---
foundation:
  topic: Kubernetes Security
  group: Network Security
  slug: overview
  image: assets/network-security.svg
  order: 60
  title: "Overview"
  description: "The default open pod network and why controlling traffic between workloads matters."
  publish: true
---

# Overview

The Kubernetes network model has one rule that surprises everyone the first time: **every pod can reach every other pod, in every namespace, with no NAT and no firewall in between.**

That flat network is what makes Kubernetes networking simple to reason about. It is also a gift to attackers. Compromise one pod (a forgotten test deployment, an internet-facing app with a vulnerable dependency) and the entire cluster is one hop away: databases, internal admin APIs, the metrics stack, other teams' workloads.

```text
default: internet-facing pod  ->  payments database   allowed
         batch job            ->  admin dashboard     allowed
         anything             ->  anything            allowed
```

Network security in Kubernetes means replacing that default with intent: traffic flows because someone declared it should, not because nothing stopped it.

## NetworkPolicy Is The Pod Firewall

The `NetworkPolicy` resource is the built-in tool for this. It selects pods by label and declares what traffic may enter them (**ingress**) and what traffic may leave them (**egress**). Everything not allowed by some policy is dropped, once a pod is selected by any policy in that direction.

Two properties define how policies behave:

- **Policies are additive.** There is no deny rule. You deny by default and then allow specific flows; multiple policies on the same pod combine into a union of allowances.
- **Selection flips the default.** A pod selected by no ingress policy accepts everything. The moment one ingress policy selects it, only allowed traffic gets in.

## The Enforcement Catch

The API server will happily store a `NetworkPolicy`, but the **CNI plugin** on each node is what actually enforces it. A cluster whose CNI does not implement NetworkPolicy accepts your policies and silently ignores them.

This matters for the lab cluster: kind's default CNI, `kindnet`, does **not** enforce NetworkPolicy. The next page shows how to create a kind cluster with an enforcing CNI. Common enforcing choices:

| CNI | Enforcement |
| --- | --- |
| Calico | Full NetworkPolicy, plus its own richer policy CRDs |
| Cilium | Full NetworkPolicy via eBPF, plus L7 and FQDN policies |
| kindnet, flannel (bare) | Stores policies, enforces nothing |

Managed clusters vary too: some enable enforcement by default, others need it switched on. Verify with a real test (apply a deny policy and try the connection), not by reading documentation.

## Ingress And Egress Are Different Problems

The two directions protect against different failures:

- **Ingress control** limits lateral movement. The payments database accepts connections only from the payments API, so a compromised marketing pod cannot even complete a TCP handshake with it.
- **Egress control** limits what a compromised pod can do next: exfiltrate data, call a command-and-control server, download attacker tooling.

Most teams start with ingress because it maps naturally to "who may call this service". Egress is rarer and harder, and covered on its own page, but it is what turns a compromised pod from a beachhead into a dead end.

## The Zero-Trust Direction

The end state to aim for, namespace by namespace:

1. Default-deny ingress in every application namespace.
2. Default-deny egress, with an explicit allowance for DNS.
3. One policy per real traffic flow, named after the flow it allows.
4. Cluster-external exposure only through a controlled ingress path.

The next pages build exactly this: NetworkPolicy mechanics, securing traffic entering the cluster, and restricting where workloads may send traffic.
