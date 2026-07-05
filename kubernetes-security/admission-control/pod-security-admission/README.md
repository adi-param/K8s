---
foundation:
  topic: Kubernetes Security
  group: Admission Control
  slug: pod-security-admission
  image: ../assets/admission-control.svg
  order: 41
  title: "Pod Security Admission"
  description: "The built in controller that enforces the Pod Security Standards on namespaces."
  publish: true
---

# Pod Security Admission

Pod Security Admission (PSA) is the built-in admission controller that enforces the **Pod Security Standards**. It needs no installation and is configured entirely with namespace labels.

## The Three Standards

The Pod Security Standards define three levels:

| Level | Meaning |
| --- | --- |
| `privileged` | No restrictions. For system components that genuinely need host access. |
| `baseline` | Blocks known privilege escalations: privileged containers, host namespaces, hostPath volumes. |
| `restricted` | Current hardening best practice: non-root, all capabilities dropped, seccomp enabled. |

## The Three Modes

Each level can be applied in three modes, and a namespace can combine them:

- **enforce:** violating pods are rejected.
- **audit:** violations are recorded in the audit log, pods still run.
- **warn:** violations return a warning to the client, pods still run.

Audit and warn exist so you can measure the blast radius of a policy before enforcing it.

## Label A Namespace

Create a namespace and enforce the restricted standard on it:

```bash
kubectl create namespace psa-lab
kubectl label namespace psa-lab \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Try to run a plain pod:

```bash
kubectl run bare --image=busybox:1.36 --restart=Never -n psa-lab -- sleep 3600
```

Expected output:

```text
Error from server (Forbidden): pods "bare" is forbidden: violates PodSecurity
"restricted:latest": allowPrivilegeEscalation != false, unrestricted capabilities,
runAsNonRoot != true, seccompProfile
```

The error lists exactly which fields are missing. Fix them:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
  namespace: psa-lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        seccompProfile:
          type: RuntimeDefault
```

```bash
kubectl apply -f hardened-pod.yaml
kubectl get pod hardened -n psa-lab
```

Expected output:

```text
NAME       READY   STATUS    RESTARTS   AGE
hardened   1/1     Running   0          5s
```

## Pin The Standard Version

The definition of each level can gain checks as Kubernetes evolves. Pin the standard to a Kubernetes version so an upgrade cannot change policy underneath you:

```bash
kubectl label namespace psa-lab \
  pod-security.kubernetes.io/enforce-version=v1.31
```

## Roll Out Safely

For an existing namespace with running workloads, start with warn and audit only:

```bash
kubectl label namespace lab \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

Every apply now prints warnings for violating pods without breaking anything. Once the workloads are clean, add the `enforce` label.

Enforcement only happens when pods are **created**. Already-running pods are not evicted when you add labels; they are re-checked the next time they are recreated.

## Clean Up

```bash
kubectl delete namespace psa-lab
kubectl label namespace lab \
  pod-security.kubernetes.io/warn- \
  pod-security.kubernetes.io/audit-
```

## Practical Guidance

1. Enforce `restricted` on every application namespace; treat exceptions as findings to fix.
2. Use `baseline` as the floor for namespaces that cannot yet meet `restricted`.
3. Always set `warn` and `audit` to the level you plan to enforce next, so drift is visible early.
4. Pin `enforce-version` so cluster upgrades do not change enforcement silently.
5. PSA cannot express custom rules such as allowed registries or required labels. For those, add a policy engine, covered next.
