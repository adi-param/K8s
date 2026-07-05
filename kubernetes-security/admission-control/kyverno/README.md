---
foundation:
  topic: Kubernetes Security
  group: Admission Control
  slug: kyverno
  image: ../assets/admission-control.svg
  order: 42
  title: "Kyverno"
  description: "A Kubernetes native policy engine for validating, mutating, and generating resources."
  publish: true
---

# Kyverno

Kyverno is a policy engine built specifically for Kubernetes. Policies are written as Kubernetes resources in YAML, so there is no new language to learn for common rules.

Kyverno installs as a set of admission webhooks and can do three things:

- **validate:** reject or flag objects that break a rule
- **mutate:** patch objects on the way in, for example adding default labels
- **generate:** create companion resources, for example a default `NetworkPolicy` for every new namespace

## Install

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace
```

Verify the controllers are running:

```bash
kubectl get pods -n kyverno
```

Expected output:

```text
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-...                 1/1     Running   0          60s
kyverno-background-controller-...                1/1     Running   0          60s
kyverno-cleanup-controller-...                   1/1     Running   0          60s
kyverno-reports-controller-...                   1/1     Running   0          60s
```

## Write A Validation Policy

This `ClusterPolicy` rejects privileged containers. The `pattern` mirrors the shape of a pod spec; `=(...)` means "if this field is present, it must match":

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-containers
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: privileged-containers
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

Apply it and test:

```bash
kubectl apply -f disallow-privileged.yaml
kubectl run priv --image=busybox:1.36 --restart=Never -n lab \
  --overrides='{"spec":{"containers":[{"name":"priv","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

Expected output:

```text
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/lab/priv was blocked due to the following policies

disallow-privileged-containers:
  privileged-containers: 'validation error: Privileged containers are not allowed. ...'
```

An unprivileged pod still runs:

```bash
kubectl run ok --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
```

## Audit Mode And Policy Reports

Set `validationFailureAction: Audit` and Kyverno records violations instead of blocking. Because `background: true`, existing resources are scanned too:

```bash
kubectl get policyreport -A
```

The report lists pass and fail counts per namespace. This is the safe way to trial a policy against a live cluster before switching it to `Enforce`.

## Ready-Made Policies

The Kyverno project maintains a large policy library, including a full implementation of the Pod Security Standards, at [kyverno.io/policies](https://kyverno.io/policies/). Starting from the library and tightening is faster and safer than writing rules from scratch.

## Clean Up

```bash
kubectl delete pod ok -n lab
kubectl delete clusterpolicy disallow-privileged-containers
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno
```

## Practical Guidance

1. Start every policy in `Audit` mode, review the policy reports, then move to `Enforce`.
2. Keep `background: true` so violations in existing objects are visible, not just new ones.
3. Prefer validation over mutation; silent mutation surprises the people who own the manifests.
4. Match on `Pod` for runtime truth, but remember blocked deployment pods fail quietly inside the ReplicaSet, so check `kubectl describe replicaset` when a rollout stalls.
5. Kyverno itself runs with high privileges; pin its version, restrict who can edit `ClusterPolicy` objects, and monitor its webhook availability.
