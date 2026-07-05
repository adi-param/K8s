---
foundation:
  topic: Kubernetes Security
  group: Detection & Response
  slug: overview
  image: assets/detection-response.svg
  order: 100
  title: "Overview"
  description: "Assuming breach: how to see what is happening in a cluster and respond to it."
  publish: true
---

# Overview

Everything so far in this foundation has been prevention: harden the control plane, lock down RBAC, restrict what pods can do. Prevention fails eventually, whether from a zero-day, a leaked credential, or a misconfiguration that slipped past review. Detection is how you find out, and response is what you do next.

The question this section answers: **if an attacker were in your cluster right now, would anything tell you?**

## The Two Signal Planes

A Kubernetes cluster produces security signals in two distinct places:

| Plane | Source | What it shows |
| --- | --- | --- |
| Control plane | API server audit logs | Who asked the API for what: exec into pods, secrets reads, RBAC changes |
| Runtime | Kernel events (syscalls) from each node | What processes actually did: shells spawned, files written, connections opened |

They are complementary, not alternatives. An attacker using a stolen kubeconfig shows up in audit logs but may never trip a runtime rule. An attacker exploiting a vulnerable app never touches the API but spawns processes the runtime plane sees. You need both.

## What An Attack Looks Like

A typical container compromise, mapped to the signals it leaves:

1. **Initial access.** An internet-facing app is exploited. Signal: little or nothing, which is why the later stages matter.
2. **Execution.** The attacker spawns a shell or downloads a tool. Signal: runtime, whether a shell in a container, an outbound download, or a new binary executed.
3. **Credential access.** They read the service account token at `/var/run/secrets/kubernetes.io/serviceaccount/token` and query the metadata endpoint. Signal: runtime file read, then audit log entries from an unexpected identity.
4. **Discovery and lateral movement.** They enumerate the API: `list secrets`, `get pods --all-namespaces`, create workloads. Signal: audit log, a service account doing things its application never does.
5. **Persistence.** A new CronJob, a mutating webhook, a ClusterRoleBinding. Signal: audit log writes to objects that rarely change.

The pattern: runtime signals catch the beachhead, audit signals catch the spread.

## Response Basics

When something fires, the order of operations matters:

1. **Isolate, do not delete.** Deleting the pod destroys the evidence and tips the attacker off. Apply a deny-all NetworkPolicy to the pod's labels, or cordon the node:

```bash
kubectl cordon <node-name>
```

2. **Capture before you kill.** Grab pod logs, `kubectl describe` output, and the container filesystem while it still exists:

```bash
kubectl logs suspect-pod -n lab --timestamps > pod.log
kubectl get pod suspect-pod -n lab -o yaml > pod.yaml
```

3. **Rotate what the pod could reach.** Assume the service account token, any mounted secrets, and any credentials in env vars are compromised. Rotate them all.
4. **Trace the entry.** Use audit logs to walk backwards: what did this identity touch, when did the behaviour start, what else did it create?
5. **Rebuild, never clean.** Redeploy from a known-good image; do not try to disinfect a running container.

## What This Section Covers

- **Audit Logs:** turning on and reading the control-plane record.
- **Falco:** open source runtime detection from kernel events.
- **Runtime Detection:** the broader discipline, covering what to watch, the tool landscape, and how to test that your detections actually fire.

Detection without response practice is a dashboard nobody reads. The goal by the end of this section is signals you have seen fire, on a cluster you attacked yourself.
