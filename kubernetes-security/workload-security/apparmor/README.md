---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: apparmor
  image: ../assets/workload-security.svg
  order: 55
  title: "AppArmor"
  description: "Confining container behaviour with AppArmor mandatory access control profiles."
  publish: true
---

# AppArmor

AppArmor is a Linux **mandatory access control** (MAC) system. Where seccomp filters *which syscalls* a process may make, AppArmor controls *what those syscalls may touch*: which file paths can be read or written, whether the network can be used, which capabilities are honoured.

The two are complementary. Seccomp can block the `mount` syscall entirely; AppArmor can allow file writes in general but deny them under `/etc`. The `RuntimeDefault` seccomp profile plus the runtime's default AppArmor profile is a solid pair.

## Check Node Support

AppArmor is a kernel feature, and unlike seccomp it is not everywhere. Ubuntu and Debian nodes (including GKE) enable it; many other distros use SELinux instead. Check a node:

```bash
kubectl get nodes -o wide   # find a node name
docker exec k8s-security-lab-control-plane \
  cat /sys/module/apparmor/parameters/enabled
```

`Y` means the node enforces AppArmor. On a kind cluster running under Docker Desktop on macOS, the answer is usually a missing file: the VM kernel ships without AppArmor, so treat this page as a read-through and run the exercise on an Ubuntu host or a GKE cluster.

## The Profile Field

Since Kubernetes 1.30, AppArmor is a first-class `securityContext` field (older clusters used the `container.apparmor.security.beta.kubernetes.io/<container>` annotation):

```yaml
spec:
  securityContext:
    appArmorProfile:
      type: RuntimeDefault
```

| Type | Meaning |
| --- | --- |
| `Unconfined` | No AppArmor confinement |
| `RuntimeDefault` | The container runtime's default profile |
| `Localhost` | A named profile already loaded on the node |

On AppArmor-enabled nodes the runtime default is typically applied even without the field; setting it explicitly makes the expectation visible and enforceable.

## A Custom Profile

Profiles are text rules loaded into the kernel on each node. This one allows everything except file writes:

```text
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,
  network,
  capability,

  # deny all file writes
  deny /** w,
}
```

Load it on every node the pod can schedule on:

```bash
sudo apparmor_parser -r /etc/apparmor.d/k8s-deny-write
```

Reference it from the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-demo
  namespace: lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: k8s-deny-write
```

On a supporting cluster:

```bash
kubectl exec -n lab apparmor-demo -- touch /tmp/x
```

Expected output:

```text
touch: /tmp/x: Permission denied
command terminated with exit code 1
```

The write was allowed by seccomp (the `open` syscall is fine), permitted by file permissions, and then denied by AppArmor policy. If the referenced profile is *not* loaded on the node, the pod is rejected outright with `Blocked`. Kubernetes fails closed rather than running unconfined.

Distributing profiles to nodes by hand does not scale. The [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator) manages AppArmor profiles (and seccomp profiles) as cluster resources.

## Clean Up

```bash
kubectl delete pod apparmor-demo -n lab
```

## Practical Guidance

1. Know your nodes: AppArmor hardening plans mean nothing on kernels without it. Check `/sys/module/apparmor/parameters/enabled` in your actual node image.
2. Set `appArmorProfile: RuntimeDefault` explicitly on hardened workloads so the confinement is declared, not assumed.
3. Reach for custom profiles on high-exposure workloads (ingress proxies, anything parsing untrusted input) where "deny writes outside these paths" is easy to state.
4. Use the Security Profiles Operator instead of hand-copying profiles to nodes.
5. AppArmor is the last layer, not the first. Non-root, dropped capabilities, read-only root, and `RuntimeDefault` seccomp deliver more value per hour of effort. Add AppArmor once those are enforced.
