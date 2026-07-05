---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: overview
  image: assets/workload-security.svg
  order: 50
  title: "Overview"
  description: "Hardening the pod itself so a compromised container has as little power as possible."
  publish: true
---

# Overview

Workload security assumes the worst case: an attacker is already executing code inside your container. Everything in this section exists to answer one question: **how much damage can they do from there?**

A container is not a security boundary by default. It is a process on the node, sharing the node's kernel, often running as root. Workload hardening shrinks that process's power layer by layer:

| Layer | Controls | What it limits |
| --- | --- | --- |
| Security context | `runAsNonRoot`, `allowPrivilegeEscalation`, `privileged` | Who the process is |
| Linux capabilities | `capabilities.drop`, `capabilities.add` | Which privileged kernel operations it may perform |
| Read-only root filesystem | `readOnlyRootFilesystem` | Whether it can tamper with its own image |
| Seccomp | `seccompProfile` | Which syscalls it may make |
| AppArmor | `appArmorProfile` | Which files, network, and capabilities the kernel lets it touch |

Each layer catches what the previous one missed. A non-root process can still use dangerous syscalls; seccomp blocks those. A syscall filter cannot stop writes to a mounted path; a read-only filesystem can.

## The Hardened Baseline

This spec is the practical target for most application pods. It is also exactly what the `restricted` Pod Security Standard enforces:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

Very few applications actually need more than this. When one does, the fix is to add back the single capability or writable path it needs, not to remove the hardening.

## Order Of Adoption

If you are hardening existing workloads, this order gives the most protection for the least breakage:

1. **Run as non-root.** Stops the largest class of container escapes outright.
2. **Set `allowPrivilegeEscalation: false` and drop all capabilities.** Cheap, rarely breaks anything.
3. **Enable the `RuntimeDefault` seccomp profile.** Blocks dozens of exotic syscalls almost no application uses.
4. **Make the root filesystem read-only.** Needs a quick check of where the app writes; mount `emptyDir` there.
5. **Add AppArmor profiles** where your nodes support them.

Admission control from the previous section is how these settings become mandatory rather than optional: enforce the `restricted` standard and the hardened baseline stops being a convention and becomes policy.

The next pages walk through each layer with hands-on examples.
