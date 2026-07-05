---
foundation:
  topic: Kubernetes Security
  group: Summary
  slug: next-learning-path
  image: ../assets/summary.svg
  order: 112
  title: "Next Learning Path"
  description: "Where to go after this foundation to keep building your security skills."
  publish: true
---

# Next Learning Path

You now have the core of cluster security: identity, authorization, admission, workload hardening, network policy, secrets, hardening, supply chain, and detection. This page is about what to do with that foundation next.

## First, Make It Stick

Reading is not retention. Before starting anything new, turn this foundation into something durable:

- Rebuild the `kind` lab from scratch and re-run the exercises without looking at the pages. The ones you have to peek at are the ones you have not learned yet.
- Apply one control to a real cluster you own, even just enforcing `restricted` Pod Security on one namespace. The gap between the lab and reality is where the actual learning is.
- Write down, for your own clusters, which checklist items are green and which are not. That list is your backlog.

## Three Directions

Where you go next depends on what you want to become.

### 1. Platform / Defensive Depth

Go deeper on running secure clusters at scale.

- **Service mesh security**: mTLS between services, identity-based authorization with Istio or Linkerd. This is the natural next layer above NetworkPolicy.
- **GitOps and policy as code**: manage RBAC, network policy, and admission rules through Argo CD or Flux so security config is reviewed and versioned like application code.
- **Multi-tenancy**: hard vs soft isolation, virtual clusters, and where namespaces stop being enough.
- **Managed platform specifics**: the shared responsibility model on EKS, GKE, or AKS, and the cloud IAM ↔ Kubernetes RBAC boundary.

### 2. Offensive / Red Team

Learn to break clusters so you defend them better.

- Work through [Bad Pods](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation) hands-on: build each pod, escalate to the node, then write the admission policy that blocks it.
- Deliberately vulnerable environments: [Kubernetes Goat](https://madhuakula.com/kubernetes-goat/) and [kube-hunter](https://github.com/aquasecurity/kube-hunter) in `--active` mode against a throwaway cluster.
- Container escape research: read the runc and containerd CVE write-ups and understand why each hardening layer in the workload section exists.

### 3. Supply Chain / Software Integrity

Extend from the cluster back into how software reaches it.

- **Sigstore in depth**: keyless signing, transparency logs, and enforcing verification at admission.
- **SLSA levels**: take a pipeline from unsigned builds to verifiable provenance, level by level.
- **Dependency and build security**: reproducible builds, SBOM diffing, and detecting tampering in the pipeline itself.

## Certifications, If You Want Them

Not required, but useful for structure and for signalling:

- **CKS (Certified Kubernetes Security Specialist)**: the direct next step; maps almost exactly onto this foundation and is hands-on. Requires a current CKA.
- **CKA**: worth it first if your general Kubernetes operations are shaky; security sits on top of solid fundamentals.

## A Suggested Order

If you want a single path rather than a menu:

1. Cement this foundation on a real cluster.
2. Do the CKS curriculum; it forces breadth and speed.
3. Spend a month on the offensive track; it makes every defensive control obvious.
4. Then specialise in whichever of the three directions matches your role.

Security is a moving target, not a finish line. The habit that matters most is the recurring one: scan, read the release notes, fix the worst finding, and prevent its regression. Do that on a real cluster every month and you will outpace any certificate.
