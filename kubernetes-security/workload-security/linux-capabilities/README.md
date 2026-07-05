---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: linux-capabilities
  image: ../assets/workload-security.svg
  order: 52
  title: "Linux Capabilities"
  description: "Dropping and adding the fine grained kernel privileges a container is granted."
  publish: true
---

# Linux Capabilities

Linux splits the traditional all-or-nothing power of root into about forty **capabilities**, distinct privileges like binding low ports, sending raw packets, or changing file ownership. A process can hold some capabilities and not others.

Container runtimes grant every container a default set of around a dozen capabilities, enough to keep old images working, and more than almost any application needs. Workload hardening means dropping all of them and adding back only what the app truly requires.

## See The Defaults

```bash
kubectl run caps-default --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
kubectl exec -n lab caps-default -- grep CapEff /proc/1/status
```

Expected output:

```text
CapEff: 00000000a80425fb
```

That bitmask decodes to the runtime's default set: `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `NET_BIND_SERVICE`, `NET_RAW`, `KILL`, and a few more.

`NET_RAW` is a concrete example of unnecessary power: it allows crafting raw packets, which is useful for `ping` and for ARP spoofing attacks against neighbouring pods. Prove the pod has it:

```bash
kubectl exec -n lab caps-default -- ping -c1 127.0.0.1
```

Expected output:

```text
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.049 ms
```

## Drop Everything

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: caps-none
  namespace: lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

```bash
kubectl apply -f caps-none.yaml
kubectl exec -n lab caps-none -- grep CapEff /proc/1/status
kubectl exec -n lab caps-none -- ping -c1 127.0.0.1
```

Expected output:

```text
CapEff: 0000000000000000
ping: permission denied (are you root?)
```

The process runs fine. It just can no longer perform privileged kernel operations. Most applications never notice the difference.

## Add Back Only What Is Needed

The classic legitimate need: a server binding port 80 or 443 as a non-root user.

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

This is the pattern to copy: `drop: ["ALL"]` always, then a short explicit `add` list that reviewers can question. Note that the `restricted` Pod Security Standard allows exactly this shape: drop all, add back `NET_BIND_SERVICE` only.

## Capabilities To Never Grant Lightly

| Capability | Why it is dangerous |
| --- | --- |
| `SYS_ADMIN` | A grab bag of mount, namespace, and device powers; close to full root |
| `NET_ADMIN` | Reconfigure interfaces, routing, and firewall rules |
| `SYS_PTRACE` | Inspect and inject into other processes |
| `NET_RAW` | Craft raw packets; enables spoofing attacks inside the pod network |
| `DAC_READ_SEARCH` / `DAC_OVERRIDE` | Bypass file permission checks |
| `SYS_MODULE` | Load kernel modules, instant node compromise |

If a workload asks for one of these, the question is not "how do I grant it" but "why does this design need it".

## Clean Up

```bash
kubectl delete pod caps-default caps-none -n lab
```

## Practical Guidance

1. `drop: ["ALL"]` on every container, without exception.
2. Add capabilities back one at a time, each with a reason a reviewer can check.
3. Fix low-port binding with `NET_BIND_SERVICE`, or avoid it entirely by listening on a port above 1024 behind a Service.
4. Remember capabilities interact with user identity: dropping capabilities while running as root still leaves root's file access. Combine with `runAsNonRoot`.
5. Enforce the drop-ALL pattern with admission policy so it cannot regress one manifest at a time.
