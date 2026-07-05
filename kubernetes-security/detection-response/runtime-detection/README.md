---
foundation:
  topic: Kubernetes Security
  group: Detection & Response
  slug: runtime-detection
  image: ../assets/detection-response.svg
  order: 103
  title: "Runtime Detection"
  description: "Spotting and stopping malicious behaviour in running workloads."
  publish: true
---

# Runtime Detection

Falco is one tool. Runtime detection is the discipline: watching what workloads actually *do*, deciding what "normal" looks like, and noticing the difference. This page zooms out to the signals, the tool landscape, and, most importantly, how to prove your detections work.

## What To Watch

Four behaviour families cover most container attacks:

| Signal | Normal | Suspicious |
| --- | --- | --- |
| Process lineage | App spawns its known helpers | App spawns `sh`, `curl`, `wget`, or a package manager |
| Network connections | Known service-to-service calls | The metadata endpoint, mining pools, connect-back to raw IPs |
| File activity | Reads its config, writes its scratch dir | Reads `/etc/shadow` or the SA token, writes new binaries |
| Privilege changes | Steady identity | `setuid`, new capabilities, mount attempts |

A compromised container almost always trips one of these early. The attacker's first move after code execution is to look around (spawn a shell), pull tools (network + file write), or grab credentials (read the token), all visible at runtime.

## The Enabling Technology: eBPF

Modern runtime detection is built on **eBPF**: small programs the kernel runs at syscall and network hooks, streaming events to user space with low overhead and no kernel module to maintain. It is why a single node agent can see every process, connection, and file open across every container without instrumenting the workloads themselves.

## The Tool Landscape

| Tool | Model | Can enforce? |
| --- | --- | --- |
| Falco | eBPF events → rules → alerts | No (detect only) |
| Tetragon | eBPF, kernel-level policy | Yes, can kill a process inline |
| KubeArmor | eBPF + LSM (AppArmor/BPF-LSM) | Yes, block file/process/network |
| Commercial CNAPP / container EDR | Agent + managed rule content and UI | Usually yes |

The split that matters: **detect-only** tools tell you after the fact; **enforcing** tools (Tetragon, KubeArmor) can terminate the offending process synchronously. Enforcement is powerful and dangerous, because a bad rule kills legitimate workloads, so most teams start in detect mode and graduate specific, high-confidence rules to enforce.

## A Detection Content Strategy

Do not try to detect "everything". Start from known TTPs, the moves attackers actually make in containers:

1. **Interactive shell in a container** (`sh`/`bash` under a non-shell parent): post-exploitation.
2. **Reverse shell**: a shell process with a socket on stdin/stdout, or `/dev/tcp` connect-backs.
3. **Cloud metadata access**: connections to `169.254.169.254` from a pod.
4. **Credential reads**: opens of `/var/run/secrets/.../token` or `/etc/shadow`.
5. **Cryptominer indicators**: connections to mining pools, sustained max CPU, known miner process names.
6. **`kubectl` from inside a pod**: an app container should not be talking to the API server interactively.

Each is a small, high-signal rule. Together they catch the common paths without drowning you in noise.

## Map To MITRE ATT&CK For Containers

MITRE maintains an ATT&CK matrix for Containers. Tagging each detection with its technique (e.g. `T1059` Command and Scripting Interpreter, `T1552` Unsecured Credentials, `T1610` Deploy Container) turns a pile of rules into a coverage map: you can see which tactics you can spot and which you are blind to. Falco and the commercial tools ship ATT&CK-tagged rule sets you can start from.

## Test Your Detections

A detection you have never seen fire is a hypothesis, not a control. Validate coverage by running the attacker actions yourself, **on your own lab cluster**, and confirming each one produces an alert. This is authorized self-testing of the `k8s-security-lab` cluster, the same discipline as atomic red team tests.

```bash
kubectl run redteam --image=busybox:1.36 --restart=Never -n lab -- sleep 3600

# 1. Interactive shell, expect a "shell spawned" detection
kubectl exec -n lab redteam -- sh -c "echo shell test"

# 2. Read the service account token, expect a credential-access detection
kubectl exec -n lab redteam -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# 3. Reach the cloud metadata endpoint, expect a network detection
kubectl exec -n lab redteam -- wget -qO- --timeout=2 http://169.254.169.254/ || true
```

After each command, check your detection tool for the matching alert:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
```

Three actions, three alerts. If any action is silent, you have found a gap: write the rule, run the test again, and confirm it fires. Re-run this suite after every tool upgrade; detectors break quietly.

## A Response Playbook Sketch

When a runtime alert fires and you believe it is real:

1. **Isolate:** deny-all NetworkPolicy on the pod's labels, or cordon the node. Do not delete.
2. **Capture:** logs, `describe`, pod YAML, and if the tool supports it, the process tree and connections it recorded.
3. **Rotate:** the pod's service account token and every secret it mounted.
4. **Investigate:** pivot to audit logs to see what the identity did on the API.
5. **Rebuild:** redeploy from a trusted image; never disinfect a live container.

## Clean Up

```bash
kubectl delete pod redteam -n lab
```

## Practical Guidance

1. Detect first, enforce later; prove a rule is clean in observe mode before letting a tool kill on it.
2. Build detection content from real TTPs and tag it to MITRE ATT&CK so your blind spots are visible.
3. Test every detection by performing the action yourself on the lab cluster; untested detections rot.
4. Correlate the two planes: a runtime shell alert plus audit-log API calls from the same pod is a confirmed incident, not a maybe.
5. Write the response steps down before an incident; deciding how to isolate a pod at 3am is how evidence gets deleted.
