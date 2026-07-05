---
foundation:
  topic: Kubernetes Security
  group: Cluster Hardening
  slug: api-server
  image: ../assets/cluster-hardening.svg
  order: 91
  title: "API Server"
  description: "Hardening the front door to the cluster: flags, anonymous access, and audit configuration."
  publish: true
---

# API Server

The API server is the only component clients talk to. Its command-line flags decide who can authenticate, how requests are authorized, which admission plugins run, and what gets logged. A single wrong flag here undoes every policy built on top of it.

## Inspect The Real Flags

On kubeadm-based clusters (including kind), the API server runs as a **static pod** whose manifest lives on the control plane node. Read it directly:

```bash
docker exec k8s-security-lab-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

The `command:` list is the security configuration. The flags below are the ones worth checking first.

## Anonymous Authentication

```bash
docker exec k8s-security-lab-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep anonymous
```

If the flag is absent, note that `--anonymous-auth` **defaults to `true`**. Unauthenticated requests are not rejected; they become the user `system:anonymous` in the group `system:unauthenticated`.

That sounds alarming but is survivable *only because* RBAC grants that group almost nothing, just discovery endpoints like `/healthz` and `/version`. Confirm what anonymous can actually do:

```bash
kubectl auth can-i get pods --as=system:anonymous
```

Expected output:

```text
no
```

The danger is the combination: anonymous auth enabled *plus* a careless ClusterRoleBinding to `system:unauthenticated` equals an open cluster. Real-world breaches have started exactly this way. Either set `--anonymous-auth=false` (if nothing unauthenticated needs `/healthz`) or treat bindings to `system:unauthenticated` and `system:authenticated` as incidents.

## Authorization Mode

```bash
docker exec k8s-security-lab-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
```

Expected output:

```text
    - --authorization-mode=Node,RBAC
```

- `Node` scopes each kubelet to its own node's resources.
- `RBAC` is everything the Authorization section covered.
- `AlwaysAllow` must never appear here. Older tutorials used it; it turns off authorization entirely.

## Admission Plugins

```bash
docker exec k8s-security-lab-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins
```

Expected output:

```text
    - --enable-admission-plugins=NodeRestriction
```

`NodeRestriction` (paired with the `Node` authorizer) stops a compromised kubelet from modifying other nodes' objects or labeling its own node to attract sensitive workloads. Defaults cover the rest (`PodSecurity`, `ServiceAccount`, and others are on by default in current versions).

## Audit Logging

Audit logs are the record of *who did what*, indispensable during an incident, and **off by default**. kind clusters ship without them, which you can confirm:

```bash
docker exec k8s-security-lab-control-plane \
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep audit
```

No output means no audit trail. Enabling it takes two flags plus a policy file on the control plane node:

```text
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

The Detection & Response section covers writing the policy and reading the logs. On managed clusters this is a checkbox (CloudTrail/Cloud Logging integration). Turn it on.

## A Note On History: Insecure Ports

Until Kubernetes 1.20, the API server could serve on an **insecure port** (`--insecure-port=8080`): no TLS, no authentication, no authorization. It was removed entirely in 1.24. If you ever work on an old cluster and see that flag non-zero, that is finding number one.

## Network Exposure

Flags aside, the biggest real-world API server risk is reachability. Hundreds of thousands of API servers respond on the public internet; each one is a standing invitation for credential stuffing and CVE exploitation.

1. On managed clusters, use a **private endpoint**, or at minimum restrict the public endpoint to known CIDR ranges.
2. Reach private endpoints through a VPN or bastion, not by opening the firewall.
3. Remember the API server is a target even when authentication is perfect, because auth bypass CVEs happen. Not being reachable is the only mitigation that survives them.

## Clean Up

Nothing to clean up. This lab only read configuration.

## Practical Guidance

1. Read your API server's live flags rather than trusting documentation about defaults; versions and distributions differ.
2. Ensure `--authorization-mode` includes `Node,RBAC` and never `AlwaysAllow`.
3. Keep `NodeRestriction` in the enabled admission plugins.
4. Audit any RBAC binding that mentions `system:unauthenticated` or `system:anonymous`; there should be essentially none beyond the built-in discovery role.
5. Enable audit logging before you need it; it cannot be enabled retroactively for an incident that already happened.
6. Treat API server network exposure as the first control: private endpoint if possible, CIDR-restricted if not.
