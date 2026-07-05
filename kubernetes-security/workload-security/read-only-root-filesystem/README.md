---
foundation:
  topic: Kubernetes Security
  group: Workload Security
  slug: read-only-root-filesystem
  image: ../assets/workload-security.svg
  order: 53
  title: "Read-only Root Filesystem"
  description: "Preventing writes to the container filesystem to blunt tampering and persistence."
  publish: true
---

# Read-only Root Filesystem

`readOnlyRootFilesystem: true` mounts the container's own filesystem read-only. The process can read everything the image contains but modify none of it.

This removes a whole class of post-compromise moves. An attacker who gets code execution in the container can no longer:

- drop tools like a crypto miner or reverse shell binary onto disk
- overwrite the application's binaries, scripts, or configuration
- persist anything across a container restart

It also enforces a good architectural habit: containers should be immutable artifacts, with all real state in volumes or external stores.

## See It Work

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rofs
  namespace: lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

```bash
kubectl apply -f rofs.yaml
kubectl exec -n lab rofs -- touch /evil
```

Expected output:

```text
touch: /evil: Read-only file system
command terminated with exit code 1
```

Even `/tmp` is read-only now:

```bash
kubectl exec -n lab rofs -- touch /tmp/scratch
```

```text
touch: /tmp/scratch: Read-only file system
```

## Give The App Its Writable Paths Back

Most applications write *somewhere*: temp files, caches, pid files, logs. The pattern is a read-only root plus explicit `emptyDir` mounts at exactly those paths:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rofs-tmp
  namespace: lab
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

```bash
kubectl apply -f rofs-tmp.yaml
kubectl exec -n lab rofs-tmp -- touch /tmp/scratch && echo write ok
```

Expected output:

```text
write ok
```

An `emptyDir` lives only as long as the pod, so this keeps the immutability story intact: nothing written there survives rescheduling.

## Finding The Paths An App Needs

Turn the setting on in a dev environment and read the crash. Errors like `EROFS: read-only file system, open '/var/cache/app/...'` tell you exactly which paths need an `emptyDir`. Common ones:

- `/tmp`: almost everything
- `/var/run` or `/run`: pid files and sockets
- `/var/cache/<app>` and `/var/log/<app>`: servers like nginx
- `$HOME/.cache`: runtimes and CLIs

Two or three mounts usually cover it.

## Clean Up

```bash
kubectl delete pod rofs rofs-tmp -n lab
```

## Practical Guidance

1. Make `readOnlyRootFilesystem: true` the default in your pod templates and treat writable roots as exceptions with a reason.
2. Mount `emptyDir` volumes at the specific paths an app writes, with `sizeLimit` set so a compromised pod cannot fill the node disk.
3. Ship logs to stdout/stderr instead of files, and state to volumes or external stores, and then the read-only root costs nothing.
4. Note that the `restricted` Pod Security Standard does not require this field, so add it to your admission policy explicitly if you want it enforced.
5. Combine with `runAsNonRoot` and dropped capabilities; each control blocks the workarounds for the others.
