---
foundation:
  topic: Kubernetes Security
  group: Summary
  slug: kubernetes-security-checklist
  image: ../assets/summary.svg
  order: 110
  title: "Kubernetes Security Checklist"
  description: "A consolidated checklist covering every area in this foundation."
  publish: true
---

# Kubernetes Security Checklist

Everything in this foundation, condensed into one list. Use it two ways: as a review pass over an existing cluster, or as an acceptance checklist for a new one.

The items are ordered the way the foundation is: identity first, then what identities can do, then what gets admitted, then how workloads run, and finally the platform around them.

## Identity

- [ ] Human users authenticate through OIDC, not shared client certificates.
- [ ] Client certificates, if used at all, are short-lived and issued per person.
- [ ] Every workload has its own service account; nothing runs as `default`.
- [ ] `automountServiceAccountToken: false` unless the pod actually calls the API.
- [ ] Service account tokens are bound, audience-scoped projected tokens, not legacy secrets.

## Authorization

- [ ] RBAC grants follow least privilege: minimal verbs, minimal resources, no `*`.
- [ ] Applications get namespaced `Role`/`RoleBinding`, not ClusterRoles.
- [ ] `ClusterRoleBinding` list is short, reviewed, and every entry has an owner.
- [ ] Access to `secrets`, `pods/exec`, token creation, and impersonation is treated as admin access.
- [ ] Escalation paths are audited: who can create pods, bind roles, or edit webhooks.
- [ ] Permissions are verified with `kubectl auth can-i --as` as part of change review.

## Admission Control

- [ ] Every application namespace carries Pod Security Admission labels.
- [ ] `restricted` is enforced where possible; `baseline` is the floor everywhere else.
- [ ] `enforce-version` is pinned so upgrades cannot change policy silently.
- [ ] Custom guardrails (allowed registries, required labels, resource limits) exist as `ValidatingAdmissionPolicy` or a policy engine.
- [ ] New policies roll out warn/audit first, enforce second.
- [ ] Webhook failure policies are deliberate: you know what happens when the policy engine is down.

## Workload Security

- [ ] Pods run as non-root: `runAsNonRoot: true`, images built with a non-root `USER`.
- [ ] `allowPrivilegeEscalation: false` everywhere.
- [ ] Capabilities: `drop: ["ALL"]`, with any `add` short and justified.
- [ ] `privileged: true` appears only in system namespaces, each use reviewed.
- [ ] `readOnlyRootFilesystem: true` with `emptyDir` mounts for writable paths.
- [ ] Seccomp `RuntimeDefault` on every pod.
- [ ] AppArmor `RuntimeDefault` where nodes support it.

## Network Security

- [ ] Every namespace has a default-deny ingress policy; workloads opt in to traffic.
- [ ] Egress is restricted for sensitive workloads; DNS is the only universal allowance.
- [ ] Ingress to the cluster terminates TLS with managed certificates.
- [ ] The API server and kubelet are not exposed to networks that do not need them.

## Secrets Management

- [ ] etcd encryption at rest is enabled; Secrets are not plaintext in backups.
- [ ] RBAC on Secrets is tighter than on anything else; `list secrets` is admin-only.
- [ ] Secrets come from an external manager (External Secrets, CSI driver) rather than living in git.
- [ ] Nothing sensitive is in ConfigMaps, env dumps, or image layers.
- [ ] Rotation is possible without redeploying by hand.

## Cluster Hardening

- [ ] Anonymous auth is off on the API server and kubelet.
- [ ] kubelet authorization mode is `Webhook`, not `AlwaysAllow`.
- [ ] etcd is TLS-secured, peer and client, and reachable only from control-plane hosts.
- [ ] A CIS benchmark scan (kube-bench) runs regularly and regressions are triaged.
- [ ] Control plane and nodes run a supported Kubernetes version with a patch cadence.

## Supply Chain Security

- [ ] Images are scanned for vulnerabilities in CI and blocking thresholds exist.
- [ ] Images are pinned by digest or signed; signatures verified at admission.
- [ ] SBOMs are generated at build and stored where responders can query them.
- [ ] Base images are minimal (distroless or slim) and rebuilt on a schedule.
- [ ] Only trusted registries are admitted into the cluster.

## Detection And Response

- [ ] API server audit logging is on, with a policy that captures writes and secret access.
- [ ] Audit logs ship somewhere the cluster admin cannot quietly edit.
- [ ] Runtime detection (Falco or equivalent) watches for shells, mounts, and crypto-mining patterns.
- [ ] Alerts route to humans with enough context to act.
- [ ] You have rehearsed one incident: isolate a pod, capture evidence, rotate credentials.

## How To Use This List

Do not try to turn the whole list green in one quarter. Pick the row with the worst finding, fix it across the fleet, add an admission policy or scan so it cannot regress, and move to the next. The controls that prevent regressions are worth more than the one-time cleanups.
