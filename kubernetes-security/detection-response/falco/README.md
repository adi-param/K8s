---
foundation:
  topic: Kubernetes Security
  group: Detection & Response
  slug: falco
  image: ../assets/detection-response.svg
  order: 102
  title: "Falco"
  description: "Runtime threat detection from kernel events using Falco rules."
  publish: true
---

# Falco

Falco is the standard open source tool for the runtime signal plane. It watches **syscalls** (the same events seccomp filters) but instead of blocking them, it matches them against rules and raises alerts: a shell spawned in a container, a sensitive file read, an unexpected outbound connection.

Falco runs as a DaemonSet. On each node, a driver (modern eBPF by default) streams kernel events to the Falco engine, which enriches them with container and Kubernetes metadata so a rule can say "in a container, in this namespace" rather than just "on this host".

## Install

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace
kubectl get pods -n falco
```

Expected output:

```text
NAME          READY   STATUS    RESTARTS   AGE
falco-xxxxx   2/2     Running   0          90s
```

One pod per node. On the single-node lab cluster, one pod.

## Trigger A Detection

Falco ships with a curated default ruleset. The classic one to see first: a shell spawned inside a running container, the signature move of stage two of an attack.

```bash
kubectl run falco-demo --image=busybox:1.36 --restart=Never -n lab -- sleep 3600
kubectl exec -n lab falco-demo -- sh -c "id"
```

Watch Falco react:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=5
```

Expected output (trimmed):

```text
Notice A shell was spawned in a container with an attached terminal
(user=root container=falco-demo shell=sh parent=containerd-shim
cmdline=sh -c id k8s.ns=lab k8s.pod=falco-demo)
```

Nothing was blocked. The exec worked. Falco observed it and told you. Detection, not prevention.

## Anatomy Of A Rule

Rules are YAML: a condition over event fields, and an output template for the alert.

```yaml
- rule: Read Service Account Token
  desc: Detect a process reading the mounted service account token
  condition: >
    open_read and container
    and fd.name startswith /var/run/secrets/kubernetes.io/serviceaccount
    and not proc.name in (known_token_readers)
  output: >
    Service account token read (user=%user.name proc=%proc.name
    file=%fd.name %container.info)
  priority: WARNING

- list: known_token_readers
  items: [app-server]
```

The pieces:

- **condition:** a filter expression over syscall event fields (`fd.name`, `proc.name`, `container.*`, `k8s.*`)
- **output:** the alert text, with fields interpolated
- **priority:** EMERGENCY down to DEBUG
- **lists and macros:** named fragments that keep conditions readable and exceptions explicit

Custom rules go in the Helm value `customRules` or a mounted rules file, and layer on top of the defaults.

## Routing Alerts

Logs on a node are not a detection system. **Falcosidekick** fans Falco events out to real destinations: Slack, PagerDuty, a SIEM, a webhook.

```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/..."
```

## Living With The Noise

Honesty about running Falco in production: the default rules will fire on legitimate behaviour. Ops tools exec into containers; package managers read files that look sensitive; init containers do strange things once. Expect a tuning period:

- Prefer adding exceptions (lists, macros) over disabling whole rules.
- Tune per-rule, with the exception named after why it exists.
- Route low-priority rules to a log index, not a pager.
- Review the exceptions periodically; an exception list is also an attacker's allowlist.

## Clean Up

```bash
kubectl delete pod falco-demo -n lab
helm uninstall falco -n falco
kubectl delete namespace falco
```

## Practical Guidance

1. Deploy Falco (or an equivalent) before you think you need it; runtime signals cannot be collected retroactively either.
2. Ship events somewhere durable via Falcosidekick from day one; node logs disappear with the node.
3. Start with the default rules, run in observe mode for two weeks, and tune exceptions before paging anyone.
4. Write custom rules for your crown jewels: token reads, writes under your app's config paths, unexpected children of your app process.
5. Test your rules on purpose (exec a shell, read the token) after every upgrade; a detector that no longer fires fails silently.
