---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: seccomp
  image: ../assets/workload-security.svg
  order: 54
  title: "Seccomp"
  description: "Restricting the syscalls a container may make with seccomp profiles."
  publish: true
---

# Seccomp

Every action a container takes (opening files, sending packets, spawning processes) ends up as a **syscall** into the shared node kernel. The kernel exposes hundreds of syscalls; a typical application uses well under a hundred. The rest are pure attack surface, and kernel vulnerabilities are regularly reachable only through obscure syscalls.

Seccomp (secure computing mode) filters which syscalls a process may make. Kubernetes exposes it per pod or per container through `seccompProfile`:

| Type | Meaning |
| --- | --- |
| `Unconfined` | No filter. The historical default. |
| `RuntimeDefault` | The container runtime's curated profile, blocks 50+ dangerous or obscure syscalls |
| `Localhost` | A custom JSON profile from the node's seccomp directory |

## Enable RuntimeDefault

This is the highest-value, lowest-effort setting on this page:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-default
  namespace: lab
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

Verify the filter is active:

```bash
kubectl apply -f seccomp-default.yaml
kubectl exec -n lab seccomp-default -- grep Seccomp /proc/1/status
```

Expected output:

```text
Seccomp:        2
Seccomp_filters:        1
```

Mode `2` means a filter is loaded (`0` is unconfined). The runtime's default profile blocks syscalls like `mount`, `reboot`, `init_module`, and `ptrace` variants, things applications do not do but exploits love.

## Write A Custom Profile

`Localhost` profiles let you go tighter. This profile allows everything except the `chmod` family:

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["chmod", "fchmod", "fchmodat"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

The kubelet loads profiles from `/var/lib/kubelet/seccomp` on each node. On the kind lab cluster, place it there through the node container:

```bash
docker exec k8s-security-lab-control-plane mkdir -p /var/lib/kubelet/seccomp
docker cp deny-chmod.json k8s-security-lab-control-plane:/var/lib/kubelet/seccomp/deny-chmod.json
```

Reference it by path relative to that directory:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-custom
  namespace: lab
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: deny-chmod.json
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f seccomp-custom.yaml
kubectl exec -n lab seccomp-custom -- chmod 777 /tmp
```

Expected output:

```text
chmod: /tmp: Operation not permitted
command terminated with exit code 1
```

The syscall was refused by the kernel before it did anything.

Real allowlist profiles (default deny, explicit allow list) are stronger but must be generated from observation. The [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator) can record what a workload actually calls and distribute the resulting profile across nodes.

## Clean Up

```bash
kubectl delete pod seccomp-default seccomp-custom -n lab
docker exec k8s-security-lab-control-plane rm /var/lib/kubelet/seccomp/deny-chmod.json
```

## Practical Guidance

1. Set `seccompProfile: RuntimeDefault` on every pod today; breakage is rare and the win is large.
2. Enforce it via the `restricted` Pod Security Standard, which requires exactly this field.
3. Reserve custom `Localhost` profiles for high-risk, well-understood workloads; they must be distributed to every node the pod can schedule on.
4. Generate allowlist profiles from recordings rather than writing them by hand.
5. When a workload crashes under a profile with errors like `Operation not permitted`, compare with `Unconfined` in dev first, because that isolates seccomp as the cause before you start editing profiles.
