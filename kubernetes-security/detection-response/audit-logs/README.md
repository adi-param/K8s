---
foundation:
  topic: Kubernetes Security
  group: Detection & Response
  slug: audit-logs
  image: ../assets/detection-response.svg
  order: 101
  title: "Audit Logs"
  description: "Configuring and reading the API server audit log to reconstruct what happened."
  publish: true
---

# Audit Logs

The API server audit log is the flight recorder of the cluster. Every request (who made it, what verb, which object, what the server answered) can be written down. When something goes wrong, this is the record you reconstruct the incident from.

Audit logging is **off by default**. A cluster without it has no control-plane memory: after an incident you will know something happened, but not who did it or what else they touched.

## What Gets Recorded

Each audit event captures the request in security-relevant detail:

- the authenticated user, their groups, and any impersonation
- the verb, resource, namespace, and object name
- the source IP and user agent
- the response code, and optionally the full request and response bodies
- timestamps for received and completed stages

## Levels And Stages

The audit **policy** decides how much is recorded per request. Four levels:

| Level | Records |
| --- | --- |
| `None` | Nothing; used to drop noise |
| `Metadata` | Who, what, when, but no bodies |
| `Request` | Metadata plus the request body |
| `RequestResponse` | Both bodies, full fidelity |

Events fire at **stages** (`RequestReceived`, `ResponseComplete`, `ResponseStarted` for watches, `Panic`). Logging only `ResponseComplete` halves volume and still tells you what happened and whether it succeeded.

## A Starter Policy

Metadata for everything, full fidelity where attackers live, and noise dropped explicitly:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]
rules:
  # High-volume, low-value reads: drop
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: ""
        resources: ["nodes", "nodes/status"]

  # Where attackers live: record everything
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets", "configmaps", "serviceaccounts/token"]
      - group: "rbac.authorization.k8s.io"
        resources: ["clusterrolebindings", "rolebindings", "clusterroles", "roles"]
  - level: RequestResponse
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Everything else: metadata
  - level: Metadata
```

Rules are evaluated top to bottom; the first match wins.

## Enable It On The Lab Cluster

The audit flags are set on the API server itself, so on kind this means a cluster config, and recreating the cluster:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-security-lab
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: ./audit-policy.yaml
        containerPath: /etc/kubernetes/audit/audit-policy.yaml
        readOnly: true
    kubeadmConfigPatches:
      - |
        kind: ClusterConfiguration
        apiServer:
          extraArgs:
            audit-policy-file: /etc/kubernetes/audit/audit-policy.yaml
            audit-log-path: /var/log/kubernetes/audit.log
            audit-log-maxage: "7"
            audit-log-maxsize: "100"
          extraVolumes:
            - name: audit
              hostPath: /etc/kubernetes/audit
              mountPath: /etc/kubernetes/audit
              readOnly: true
            - name: audit-log
              hostPath: /var/log/kubernetes
              mountPath: /var/log/kubernetes
```

```bash
kind delete cluster --name k8s-security-lab
kind create cluster --config kind-audit.yaml
```

Generate an interesting event and find it:

```bash
kubectl create namespace lab
kubectl run audit-demo --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
kubectl exec -n lab audit-demo -- id
docker exec k8s-security-lab-control-plane \
  grep '"subresource":"exec"' /var/log/kubernetes/audit.log | tail -1
```

Expected output (trimmed):

```json
{"kind":"Event","verb":"create","user":{"username":"kubernetes-admin"},
 "objectRef":{"resource":"pods","namespace":"lab","name":"audit-demo","subresource":"exec"},
 "responseStatus":{"code":101}}
```

That single line answers who exec'd into which pod, from where, and when.

## What To Alert On

A short list that catches most real attacks:

1. `pods/exec`, `pods/attach`, `pods/portforward`: interactive access to workloads.
2. `get`/`list` on secrets, especially cluster-wide lists or unusual identities.
3. Writes to RBAC objects, webhooks, and CronJobs: persistence and escalation.
4. Requests from `system:anonymous` or `system:unauthenticated` that are not health checks.
5. A service account calling APIs its application has never called before.

## Ship Logs Off The Node

A log an attacker can delete is not evidence. Forward audit logs to storage outside the cluster: a log pipeline (Fluent Bit, Vector) tailing `audit-log-path`, or the `--audit-webhook-config-file` flag pushing events directly to a collector. Managed clusters do this for you: EKS ships audit logs to CloudWatch, GKE to Cloud Logging, but usually off by default, so turn it on.

## Clean Up

```bash
kubectl delete pod audit-demo -n lab
```

Keep the audit-enabled cluster; later pages assume signals are being recorded.

## Practical Guidance

1. Turn audit logging on before you need it; it cannot be enabled retroactively.
2. Start from the policy above, not `Metadata` for everything; the volume of an unfiltered log is what kills audit programs.
3. Record `RequestResponse` for secrets, RBAC, and exec; that is where incidents are reconstructed.
4. Ship logs off-cluster with retention measured in months, not days.
5. Alert on the short list first; a handful of high-signal alerts beats a thousand unread ones.
