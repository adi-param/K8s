---
foundation:
  topic: Kubernetes Security
  group: Admission Control
  slug: overview
  image: assets/admission-control.svg
  order: 40
  title: "Overview"
  description: "How admission controllers inspect, mutate, and reject requests after authentication and authorization but before persistence."
  publish: true
---

# Overview

Admission control is the last gate a request passes before it is stored in etcd.

Authentication proved who is asking. Authorization proved they are allowed to ask. Admission control answers a different question: **is this specific object acceptable?**

```text
request -> authentication -> authorization -> admission -> stored object
```

RBAC can say "this user may create pods". It cannot say "but not privileged pods" or "only images from our registry". That is admission control's job: policy about the *content* of objects, not the identity of the requester.

## Mutating And Validating

Admission runs in two phases, in this order:

- **Mutating admission** can change the object before it is stored. Examples: injecting a sidecar, setting a default service account token policy, adding required labels.
- **Validating admission** can only accept or reject the (already mutated) object. Examples: rejecting privileged containers, requiring resource limits.

Mutation always runs first so validators see the final object. A validator can therefore safely enforce something a mutator added.

## Built-In Controllers

The API server ships with compiled-in admission controllers. See which ones are enabled:

```bash
kubectl -n kube-system get pod -l component=kube-apiserver \
  -o jsonpath='{.items[0].spec.containers[0].command}' | tr ',' '\n' | grep admission
```

Notable built-in controllers:

- **NamespaceLifecycle:** blocks creating objects in a namespace that is terminating.
- **LimitRanger:** applies default CPU and memory limits from `LimitRange` objects.
- **ServiceAccount:** attaches service accounts and tokens to pods.
- **PodSecurity:** enforces the Pod Security Standards per namespace.
- **NodeRestriction:** stops a kubelet from modifying objects beyond its own node and pods.

## Dynamic Admission

Beyond the built-ins, Kubernetes lets you add your own logic three ways:

| Mechanism | Runs | Language |
| --- | --- | --- |
| Admission webhooks | Your own service, called over HTTPS | Anything |
| `ValidatingAdmissionPolicy` | Inside the API server | CEL expressions |
| Policy engines (Kyverno, OPA Gatekeeper) | Webhooks installed as cluster add-ons | YAML patterns or Rego |

Webhooks are the most flexible and the most dangerous: a webhook with `failurePolicy: Fail` that goes down can block all matching requests cluster-wide, and one with `failurePolicy: Ignore` silently stops enforcing. Policy engines package the webhook machinery so you only write rules.

## What To Enforce First

A practical starting set for most clusters:

1. Enforce the **restricted** Pod Security Standard on application namespaces.
2. Reject privileged containers and host namespace usage everywhere else explicitly.
3. Require images to come from registries you trust.
4. Require resource requests and limits so one workload cannot starve a node.
5. Run new policies in audit or warn mode first, then switch to enforce once the violations are cleaned up.

The next pages cover Pod Security Admission, the Kyverno and OPA Gatekeeper policy engines, and the built-in CEL based Validating Admission Policy.
