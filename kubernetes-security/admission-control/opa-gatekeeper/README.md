---
foundation:
  topic: Kubernetes Security
  group: Admission Control
  slug: opa-gatekeeper
  image: ../assets/admission-control.svg
  order: 43
  title: "OPA Gatekeeper"
  description: "Policy as code with Open Policy Agent using constraint templates and constraints."
  publish: true
---

# OPA Gatekeeper

Gatekeeper brings the Open Policy Agent (OPA) to Kubernetes admission control. Policies are written in **Rego**, a general purpose policy language, which makes Gatekeeper a good fit when the same policy toolchain must also cover Terraform, CI pipelines, or application authorization.

Gatekeeper splits every policy into two objects:

- **ConstraintTemplate:** the Rego logic plus a schema for its parameters. Applying one creates a new CRD.
- **Constraint:** an instance of that CRD that says where the rule applies and with which parameters.

One template, many constraints: write "require labels" once, then require different labels in different namespaces.

## Install

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.17.1/deploy/gatekeeper.yaml
kubectl get pods -n gatekeeper-system
```

Expected output:

```text
NAME                                             READY   STATUS    RESTARTS   AGE
gatekeeper-audit-...                             1/1     Running   0          60s
gatekeeper-controller-manager-...                1/1     Running   0          60s
```

## Write A ConstraintTemplate

This template rejects objects missing required labels. The `violation` rule fires once per problem it finds:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := {label | input.review.object.metadata.labels[label]}
          missing := {label | label := required[_]; not provided[label]}
          count(missing) > 0
          msg := sprintf("missing required labels: %v", [missing])
        }
```

## Apply A Constraint

Require an `owner` label on pods in the `lab` namespace:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-owner
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["lab"]
  parameters:
    labels: ["owner"]
```

Test it:

```bash
kubectl run unowned --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
```

Expected output:

```text
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[pods-must-have-owner] missing required labels: {"owner"}
```

A labeled pod passes:

```bash
kubectl run owned --image=busybox:1.36 --restart=Never -n lab \
  --labels=owner=adi -- sleep 3600
```

## Audit Existing Resources

Gatekeeper's audit controller re-checks the cluster every minute and stores violations on the constraint itself:

```bash
kubectl get k8srequiredlabels pods-must-have-owner -o jsonpath='{.status.totalViolations}'
```

Set `enforcementAction: dryrun` to collect audit results without blocking anything, the equivalent of Kyverno's audit mode.

## Clean Up

```bash
kubectl delete pod owned -n lab
kubectl delete k8srequiredlabels pods-must-have-owner
kubectl delete constrainttemplate k8srequiredlabels
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.17.1/deploy/gatekeeper.yaml
```

## Practical Guidance

1. Start constraints with `enforcementAction: dryrun`, watch the audit results, then switch to `deny`.
2. Reuse templates from the [Gatekeeper policy library](https://open-policy-agent.github.io/gatekeeper-library/) instead of writing Rego from day one.
3. Rego is powerful but has a real learning curve; if your policies are all Kubernetes-shaped, Kyverno's YAML patterns are usually quicker to write and review.
4. Keep templates in version control and test Rego with `opa test` in CI, exactly like application code.
5. Watch the webhook's failure policy and availability: Gatekeeper down with `failurePolicy: Fail` blocks admissions cluster-wide.
