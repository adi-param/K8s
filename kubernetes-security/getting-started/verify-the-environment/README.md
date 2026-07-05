---
foundation:
  topic: Kubernetes Security
  group: Getting Started
  slug: verify-the-environment
  image: ../assets/getting-started.svg
  order: 12
  title: "Verify the Environment"
  description: "Confirm the cluster is healthy and your kubeconfig is wired up before moving on."
  publish: true
---

# Verify the Environment

Before moving into security controls, verify that your local cluster is healthy and that your shell is pointed at the right environment.

Most mistakes in Kubernetes labs come from using the wrong context, the wrong namespace, or a cluster that is not fully ready.

## Verify the Current Context

Check the active context:

```bash
kubectl config current-context
```

Expected output:

```text
kind-k8s-security-lab
```

If you see a production, shared, or cloud cluster context, stop and switch back:

```bash
kubectl config use-context kind-k8s-security-lab
```

## Verify the Default Namespace

Check the namespace on the current context:

```bash
kubectl config view --minify --output 'jsonpath={..namespace}'; echo
```

Expected output:

```text
lab
```

If the output is empty or different, set it:

```bash
kubectl config set-context --current --namespace=lab
```

## Verify Cluster Health

Check the nodes:

```bash
kubectl get nodes
```

The node should be `Ready`.

Check core pods:

```bash
kubectl get pods -n kube-system
```

For a fresh local cluster, `Running` and `Completed` are normal. Pods stuck in `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff` need investigation before you continue.

## Verify API Access

Ask the API server for basic version information:

```bash
kubectl version
```

Then check what your current identity can do in the lab namespace:

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments
kubectl auth can-i create clusterroles
```

In a default local `kind` cluster, your user is effectively a cluster administrator. That is useful for learning because you can create and remove security controls freely. It is not the permission model you should use for day-to-day work in a shared cluster.

## Verify a Test Workload Can Run

Create a short-lived pod:

```bash
kubectl run verify --image=busybox:1.36 --restart=Never --command -- sh -c 'echo ready && sleep 5'
```

Watch it complete:

```bash
kubectl get pod verify
```

Read the logs:

```bash
kubectl logs verify
```

Expected output:

```text
ready
```

Clean it up:

```bash
kubectl delete pod verify
```

## Troubleshooting Checklist

If verification fails, check these items first:

- The container runtime is running.
- `kubectl config current-context` is `kind-k8s-security-lab`.
- The node is `Ready`.
- CoreDNS pods in `kube-system` are running.
- Your machine can pull container images.
- You are not accidentally using a restricted corporate or production kubeconfig.

Once these checks pass, the environment is ready for the rest of the Kubernetes Security foundation.
