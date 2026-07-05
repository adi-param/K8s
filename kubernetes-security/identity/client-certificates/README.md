---
foundation:
  topic: Kubernetes Security
  group: Identity
  slug: client-certificates
  image: ../assets/identity.svg
  order: 21
  title: "Client Certificates"
  description: "Authenticating to the API server with x509 client certificates, and why they are hard to revoke."
  publish: true
---

# Client Certificates

Client certificates are one way to authenticate a user or component to the Kubernetes API server. The client presents an x509 certificate, and the API server verifies that it was signed by a certificate authority it trusts.

You will often see this in local clusters and older kubeconfigs. Many managed clusters now prefer external identity systems, but certificates are still important to understand because Kubernetes control-plane components rely heavily on x509.

## How Kubernetes Reads the Identity

For client certificates, Kubernetes maps certificate fields to identity:

- The certificate **Common Name** becomes the Kubernetes username.
- The certificate **Organization** values become Kubernetes groups.

For example, a certificate with this subject:

```text
CN=alice,O=platform-team
```

authenticates as:

```text
username: alice
groups: platform-team
```

RBAC can then grant permissions to `alice` or to the `platform-team` group.

## Inspect Your Current Kubeconfig

Check whether your current context uses certificate data:

```bash
kubectl config view --raw --minify
```

Look under the current user for fields like:

```yaml
client-certificate-data: ...
client-key-data: ...
```

or file paths like:

```yaml
client-certificate: /path/to/client.crt
client-key: /path/to/client.key
```

If those fields are present, that kubeconfig identity is using a client certificate.

## Why Certificates Are Risky For Humans

Client certificates are strong credentials, but they can be awkward for day-to-day human access:

- They are often long-lived.
- They are easy to copy inside kubeconfig files.
- Kubernetes does not have a simple built-in certificate revocation workflow.
- If a certificate grants admin access, anyone with that kubeconfig has admin access until the certificate expires or the trust chain changes.

That makes client certificates a poor fit for broad human access in shared environments.

## When They Are Useful

Client certificates still make sense in some places:

- control-plane component authentication
- kubelet authentication
- short-lived bootstrap or break-glass access
- local learning clusters
- tightly managed automation where rotation is handled carefully

For regular human access, prefer an external identity provider with short-lived tokens when your platform supports it.

## Practical Guidance

Use these rules:

1. Avoid sharing kubeconfigs that contain client keys.
2. Keep certificate lifetimes short where possible.
3. Do not use client certificates as a substitute for RBAC.
4. Bind permissions to groups rather than individual certificate usernames.
5. Prefer OIDC or managed cloud identity for routine human access.

Authentication only proves who the caller is. The next step is still authorization: what that identity can do.
