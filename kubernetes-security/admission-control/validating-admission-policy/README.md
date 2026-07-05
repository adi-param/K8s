---
foundation:
  topic: Kubernetes Security
  group: Admission Control
  slug: validating-admission-policy
  image: ../assets/admission-control.svg
  order: 44
  title: "Validating Admission Policy"
  description: "In tree CEL based admission policies that need no external webhook."
  publish: true
---

# Validating Admission Policy

`ValidatingAdmissionPolicy` runs policy **inside the API server** using CEL (Common Expression Language) expressions. There is no webhook, no extra deployment, and no network hop that can fail or add latency. It is GA since Kubernetes 1.30.

Like Gatekeeper, it splits logic from placement:

- **ValidatingAdmissionPolicy:** what to check, as CEL expressions over the incoming `object`.
- **ValidatingAdmissionPolicyBinding:** where to apply it and what to do on failure (`Deny`, `Warn`, or `Audit`).

## Write A Policy

Reject privileged containers in one CEL expression:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: disallow-privileged
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c,
          !has(c.securityContext) ||
          !has(c.securityContext.privileged) ||
          c.securityContext.privileged == false)
      message: "Privileged containers are not allowed."
```

The expression must hold for the request to pass. `has()` guards optional fields, and `all()` quantifies over every container.

## Bind It

Apply the policy only to namespaces labeled for it:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: disallow-privileged-binding
spec:
  policyName: disallow-privileged
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab
```

## Test It

```bash
kubectl apply -f policy.yaml -f binding.yaml
kubectl run priv --image=busybox:1.36 --restart=Never -n lab \
  --overrides='{"spec":{"containers":[{"name":"priv","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"privileged":true}}]}}'
```

Expected output:

```text
The pods "priv" is invalid: : ValidatingAdmissionPolicy 'disallow-privileged'
with binding 'disallow-privileged-binding' denied request:
Privileged containers are not allowed.
```

An unprivileged pod is admitted normally.

## Warn And Audit First

`validationActions` supports the same staged rollout as the policy engines:

```yaml
validationActions: ["Warn", "Audit"]
```

`Warn` returns a warning to the client, `Audit` writes an audit log annotation, and neither blocks the request. Switch to `["Deny"]` once violations are gone.

## Where It Fits

| Approach | Strength | Cost |
| --- | --- | --- |
| Pod Security Admission | Zero setup, standard levels | Fixed rules only |
| ValidatingAdmissionPolicy | In-tree, fast, no add-on to operate | Validation only, CEL has expression limits |
| Kyverno / Gatekeeper | Mutation, generation, reports, big policy libraries | An extra privileged component to run and upgrade |

A reasonable progression: PSA for the pod security baseline, `ValidatingAdmissionPolicy` for a handful of custom guardrails, and a policy engine only once you need mutation, generation, or a large rule set with reporting.

## Clean Up

```bash
kubectl delete validatingadmissionpolicybinding disallow-privileged-binding
kubectl delete validatingadmissionpolicy disallow-privileged
```

## Practical Guidance

1. Prefer this over a custom webhook for simple validations; the availability failure mode of webhooks disappears entirely.
2. Keep expressions small and name each check with a clear `message`; CEL one-liners get unreadable fast.
3. Use `namespaceSelector` on the binding so platform namespaces like `kube-system` are excluded deliberately, not accidentally.
4. Roll out with `Warn` and `Audit` before `Deny`, exactly as with PSA and the policy engines.
5. Use `variables` in the policy spec to factor repeated sub-expressions when policies grow.
