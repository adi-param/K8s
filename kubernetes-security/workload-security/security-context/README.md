---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: security-context
  image: ../assets/workload-security.svg
  order: 51
  title: "Security Context"
  description: "Setting user, group, and privilege fields that control how a container runs."
  publish: true
---

# Security Context

The security context is where a pod declares who its processes run as and what privileges they hold. It exists at two levels:

- **`spec.securityContext`** (pod level): defaults for every container, plus pod-wide settings like `fsGroup`.
- **`spec.containers[].securityContext`** (container level): overrides the pod level for that container.

When both set the same field, the container level wins.

## The Fields That Matter

| Field | Safe value | What it does |
| --- | --- | --- |
| `runAsNonRoot` | `true` | Refuses to start the container if its user is root |
| `runAsUser` / `runAsGroup` | non-zero UID/GID | Forces the process identity |
| `allowPrivilegeEscalation` | `false` | Blocks gaining privileges via setuid binaries like `sudo` |
| `privileged` | `false` (default) | `true` hands the container the node, never for applications |
| `fsGroup` | as needed | Group ownership applied to mounted volumes |

`privileged: true` deserves emphasis: it disables almost every isolation mechanism at once. A privileged container can access host devices, load kernel modules, and trivially escape to the node.

## See The Default

Run a pod with no security context and check who it is:

```bash
kubectl run ctx-default --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
kubectl exec -n lab ctx-default -- id
```

Expected output:

```text
uid=0(root) gid=0(root) groups=0(root),10(wheel)
```

Root. Any file the image contains, the process can modify; any port, it can bind; combined with a container runtime bug, root inside is the first step to root on the node.

## Run As Non-Root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ctx-nonroot
  namespace: lab
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
```

```bash
kubectl apply -f ctx-nonroot.yaml
kubectl exec -n lab ctx-nonroot -- id
```

Expected output:

```text
uid=1000 gid=1000 groups=1000
```

## runAsNonRoot Is A Guarantee

`runAsUser` sets the identity; `runAsNonRoot` verifies it. If an image's default user is root and the pod sets only `runAsNonRoot: true`, the kubelet refuses to start the container:

```bash
kubectl run ctx-guard --image=busybox:1.36 --restart=Never -n lab \
  --overrides='{"spec":{"containers":[{"name":"ctx-guard","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"runAsNonRoot":true}}]}}'
kubectl get pod ctx-guard -n lab
```

Expected output:

```text
NAME        READY   STATUS                       RESTARTS   AGE
ctx-guard   0/1     CreateContainerConfigError   0          5s
```

`kubectl describe` shows the reason: `container has runAsNonRoot and image will run as root`. This is the safety net that catches images which silently expect root.

The cleanest fix is an image built with a numeric non-root `USER`, so the manifest does not need to force a UID at all.

## Volumes And fsGroup

A non-root process often cannot write to mounted volumes owned by root. `fsGroup` fixes this at the pod level: Kubernetes applies the group to volume contents so the process can write.

```yaml
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
```

## Clean Up

```bash
kubectl delete pod ctx-default ctx-nonroot ctx-guard -n lab
```

## Practical Guidance

1. Set `runAsNonRoot: true` at the pod level for every application pod; add `runAsUser` only when the image does not declare a non-root user.
2. Always set `allowPrivilegeEscalation: false`; almost nothing legitimate needs setuid escalation inside a container.
3. Treat any request for `privileged: true` as an architecture review, not a pod setting.
4. Build images with a non-root `USER` so security contexts confirm identity rather than fight the image.
5. Enforce all of this with the `restricted` Pod Security Standard rather than relying on every manifest getting it right.
