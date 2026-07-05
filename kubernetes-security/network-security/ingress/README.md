---
foundation:
  topic: Kubernetes Security
  group: Network Security
  slug: ingress
  image: ../assets/network-security.svg
  order: 62
  title: "Ingress"
  description: "Controlling and securing traffic entering the cluster."
  publish: true
---

# Ingress

Traffic entering the cluster is your real attack surface: it is the one flow that, by design, comes from people you do not control. The goal is a single, deliberate, well-instrumented front door, and nothing else listening.

## Know Your Exposure Surface

Every way traffic can enter a cluster:

| Path | Exposure |
| --- | --- |
| `ClusterIP` Service | None outside the cluster: the safe default |
| `NodePort` Service | A port on **every node's** IP, usually 30000-32767 |
| `LoadBalancer` Service | A cloud load balancer per Service |
| Ingress / Gateway API | One controlled entry point routing by host and path |
| `hostNetwork` / `hostPort` pods | The pod shares the node's network stack directly |

Audit what is currently exposed:

```bash
kubectl get svc -A | grep -v ClusterIP
kubectl get ingress -A
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.hostNetwork==true)]}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}'
```

Anything in that output is reachable from outside the pod network. Every NodePort is open on every node, including nodes that never run the pod. A forgotten NodePort from a debugging session is a classic finding.

`hostNetwork: true` deserves special suspicion: the pod bypasses Services, bypasses NetworkPolicy in many CNIs, and listens on the node itself. The `baseline` Pod Security Standard blocks it for good reason.

## One Front Door

An ingress controller (NGINX Ingress, Traefik, or a Gateway API implementation) consolidates HTTP entry into one place:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: lab
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: web-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Consolidation is a security feature in itself: one place to terminate TLS, one place for access logs, one place to add rate limits and a WAF, one component to patch when the next proxy CVE lands.

For TLS, [cert-manager](https://cert-manager.io/) automates issuing and renewing certificates (for example from Let's Encrypt) into the `web-tls` Secret, removing the expired-certificate failure mode entirely.

## Back The Front Door With NetworkPolicy

The ingress controller changes your NetworkPolicy story: application pods should accept traffic **only from the controller**, not from the world or from arbitrary pods. With default-deny ingress in place (previous page), add:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller-to-web
  namespace: lab
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 80
```

Now even a compromised pod elsewhere in the cluster cannot reach `web` directly; the only path is through the controller, where authentication, logging, and rate limits live.

## Restrict Who Reaches The Controller

The controller itself should not accept traffic from everywhere:

- Put a cloud firewall or load balancer allowlist in front of it; internal-only apps get internal-only load balancers (cloud annotations control this).
- If the source IP matters for allowlisting, set `externalTrafficPolicy: Local` on the controller's Service so the client IP survives.
- Keep the controller in its own namespace with its own restrictive policies; it is a high-value target parsing untrusted input all day.

## Clean Up

```bash
kubectl delete ingress web -n lab
kubectl delete networkpolicy allow-ingress-controller-to-web -n lab
```

## Practical Guidance

1. Audit Services regularly: everything should be `ClusterIP` except the deliberate entry points.
2. Ban NodePort and `hostNetwork` for applications via admission policy; both bypass the front door.
3. Terminate TLS at the ingress controller with cert-manager-managed certificates; redirect HTTP to HTTPS.
4. Combine the ingress controller with default-deny NetworkPolicy so it is the *only* path to application pods.
5. Treat the controller as a critical workload: pin its version, watch its CVEs, restrict its namespace, and keep its logs.
