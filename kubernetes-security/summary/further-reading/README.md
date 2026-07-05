---
foundation:
  topic: Kubernetes Security
  group: Summary
  slug: further-reading
  image: ../assets/summary.svg
  order: 111
  title: "Further Reading"
  description: "Curated references, specs, and tools to go deeper on Kubernetes security."
  publish: true
---

# Further Reading

A short, curated list. Everything here earned its place; there is no attempt to be complete.

## Primary Documentation

Start with the project's own material. It is unusually good for security topics:

- [Kubernetes Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/): the upstream counterpart to this foundation's checklist.
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/): the exact field-by-field definition of `privileged`, `baseline`, and `restricted`.
- [RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/): the reference for verbs, resources, and binding semantics.
- [Admission Controllers reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/): what is compiled into the API server and what each controller does.
- [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/): audit policy stages and levels.

## Standards And Benchmarks

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes): the de facto hardening standard; run it via kube-bench rather than reading it cover to cover.
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF): opinionated, readable, and good for justifying hardening work to management.
- [OWASP Kubernetes Top 10](https://owasp.org/www-project-kubernetes-top-ten/): the most common Kubernetes risk categories, useful for prioritising.
- [SLSA](https://slsa.dev): the supply chain security levels referenced in the provenance page.

## Threat Intelligence And Attack Technique

Reading attack write-ups is the fastest way to make defensive settings feel concrete:

- [MITRE ATT&CK Containers Matrix](https://attack.mitre.org/matrices/enterprise/containers/): attacker techniques mapped to the container lifecycle.
- [Microsoft Kubernetes Threat Matrix](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/): the original Kubernetes-specific threat matrix.
- [Hacking Kubernetes](https://www.oreilly.com/library/view/hacking-kubernetes/9781492081722/) (Martin, Hausenblas): the best single book on attacking and defending clusters.
- [Bad Pods](https://bishopfox.com/blog/kubernetes-pod-privilege-escalation): eight pod configurations and exactly how each escalates to the node. Excellent companion to the workload security section.

## Tools Worth Knowing

| Tool | What it does |
| --- | --- |
| [kube-bench](https://github.com/aquasecurity/kube-bench) | Runs the CIS benchmark against your cluster |
| [kube-hunter](https://github.com/aquasecurity/kube-hunter) | Actively probes a cluster for known weaknesses |
| [Trivy](https://github.com/aquasecurity/trivy) | Image, filesystem, and manifest scanning |
| [Kyverno](https://kyverno.io) / [Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) | Policy engines covered in the admission control section |
| [Falco](https://falco.org) | Runtime syscall-level detection |
| [Cosign](https://github.com/sigstore/cosign) | Image signing and verification |
| [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) / [rakkess](https://github.com/corneliusweig/rakkess) | Answer "who can do what" quickly |
| [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator) | Manages seccomp and AppArmor profiles as cluster resources |

## Staying Current

Kubernetes security moves fast; a small recurring habit beats a big annual catch-up:

- [Kubernetes security announcements list](https://groups.google.com/g/kubernetes-security-announce): CVE announcements; low volume, subscribe.
- [Kubernetes blog, security posts](https://kubernetes.io/blog/): feature deep dives when new security machinery ships.
- Release notes for each minor version: scan the auth, admission, and node sections; deprecations here become your migration work.

## How To Read This List

Pick by role. Running clusters: CIS benchmark via kube-bench, the NSA/CISA guide, and the announcements list. Building platform policy: the Pod Security Standards and the policy engine docs. Doing security review or red teaming: Bad Pods, the threat matrices, and *Hacking Kubernetes*.
