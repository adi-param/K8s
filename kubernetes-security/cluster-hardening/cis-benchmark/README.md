---
foundation:
  topic: Kubernetes Security
  group: Cluster Hardening
  slug: cis-benchmark
  image: ../assets/cluster-hardening.svg
  order: 94
  title: "CIS Benchmark"
  description: "Measuring a cluster against the CIS Kubernetes Benchmark with kube-bench."
  publish: true
---

# CIS Benchmark

The previous pages checked control plane flags one at a time by hand. That does not scale and it is easy to miss a setting. The **CIS Kubernetes Benchmark** is the industry checklist that captures all of them, and **kube-bench** is the tool that runs the checklist against your cluster automatically.

## What The CIS Benchmark Is

The Center for Internet Security publishes consensus hardening guides for many systems, including Kubernetes. The Kubernetes Benchmark is a few hundred numbered checks across the components in this section: API server flags, etcd permissions, kubelet config, RBAC hygiene, and file ownership on the control plane node.

Each check has a level:

- **Level 1:** practical hardening that rarely breaks anything. The baseline to aim for.
- **Level 2:** stricter, environment-dependent, may not suit every cluster.

## Run kube-bench As A Job

kube-bench runs on a node, reads the same config files you inspected by hand, and scores them. The simplest way on the kind lab cluster is to run it as a Job:

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

Wait for it to finish and read the results from the pod logs:

```bash
kubectl wait --for=condition=complete job/kube-bench --timeout=120s
kubectl logs job/kube-bench
```

## Reading The Output

kube-bench prints one line per check plus a summary:

```text
[INFO] 1 Master Node Security Configuration
[INFO] 1.2 API Server
[PASS] 1.2.1 Ensure that the --anonymous-auth argument is set to false
[FAIL] 1.2.15 Ensure that the --audit-log-path argument is set
[WARN] 1.2.30 Ensure that encryption providers are appropriately configured
...
== Summary master ==
41 checks PASS
6 checks FAIL
12 checks WARN
```

The three verdicts mean different things:

- **PASS:** the check's condition is met. Nothing to do.
- **FAIL:** kube-bench confirmed the setting is wrong. These are your real work items. Note how the `FAIL` above lines up exactly with the missing audit log from the API Server page.
- **WARN:** kube-bench **could not determine** the answer automatically. It is a prompt to check manually, not a pass and not a fail. Encryption config and manual RBAC review commonly land here.

Every check comes with a remediation snippet in the full output; scroll to the `== Remediations ==` block to see exactly which flag or file to change.

## Managed Clusters Use A Different Benchmark

On EKS, GKE, or AKS you cannot read the API server manifest, because the provider runs it. CIS publishes **provider-specific benchmarks** for this reason, and kube-bench auto-detects the platform (or you pass `--benchmark eks-1.5.0`). Those variants drop the checks the provider owns and keep the ones you still control: node configuration, RBAC, and workload settings. Running the generic benchmark on a managed cluster produces misleading FAILs for things that are the provider's responsibility.

## Do Not Chase 100 Percent Blindly

A perfect score is not the goal; a **defensible** score is. Some checks legitimately do not apply, and some remediations trade real usability for a checkbox.

1. Fix every **Level 1 FAIL** first; these are genuine, low-risk hardening wins.
2. Treat every **WARN** as a task: verify manually and record the finding, do not ignore it.
3. For **Level 2** and any check you deliberately skip, write down *why*. An exception with a documented reason is fine; a silent one is technical debt.
4. Re-run kube-bench in CI or on a schedule so configuration drift shows up as a new FAIL, not a surprise during an audit.

## Clean Up

```bash
kubectl delete job kube-bench
```

## Practical Guidance

1. Run kube-bench regularly, not once, because clusters drift as they are upgraded and reconfigured.
2. Prioritize Level 1 FAILs; they are the checks this whole section has been about.
3. Understand that WARN means "verify manually," and follow through rather than treating it as a pass.
4. Use the platform-specific benchmark on managed clusters so results reflect what you actually control.
5. Document deliberate exceptions so a defensible posture does not look like negligence to the next auditor.
6. Wire the scan into CI against a representative cluster so regressions are caught before they reach production.
