---
foundation:
  topic: Kubernetes Security
  group: Identity
  slug: oidc
  image: ../assets/identity.svg
  order: 22
  title: "OIDC"
  description: "Federating cluster login to an external identity provider with OpenID Connect tokens."
  publish: true
---

# OIDC

OpenID Connect, usually shortened to OIDC, lets Kubernetes trust identities from an external identity provider.

Instead of giving every user a long-lived client certificate, the user signs in through an identity provider. The client sends a token to the Kubernetes API server, and the API server validates that token before accepting the request.

## Why Use OIDC

OIDC is commonly used for human access because it fits normal enterprise identity workflows:

- users authenticate with an existing identity provider
- access can use MFA and conditional access rules
- users can be removed centrally
- tokens are short-lived
- Kubernetes RBAC can bind permissions to identity-provider groups

Kubernetes still does not store user accounts. It trusts the token issuer and extracts a username and groups from the token claims.

## The Basic Flow

The flow looks like this:

```text
user -> identity provider -> OIDC token -> Kubernetes API server -> RBAC
```

The API server checks that:

- the token was issued by the configured issuer
- the token audience matches what the cluster expects
- the token is not expired
- the username and group claims can be read

If the token is valid, Kubernetes treats the request as coming from that username and those groups.

## What To Bind In RBAC

Prefer binding RBAC permissions to groups, not individual users.

For example:

```yaml
subjects:
  - kind: Group
    name: platform-engineers
    apiGroup: rbac.authorization.k8s.io
```

That lets group membership live in the identity provider while Kubernetes focuses on permissions.

## What To Watch For

OIDC is powerful, but the details matter:

- Token lifetimes should be short.
- The issuer and audience must be correct.
- Group claims must be stable and predictable.
- Admin groups should be small and reviewed.
- CI/CD access should not be mixed casually with human login.
- A user leaving the company should lose access through the identity provider, not by editing many Kubernetes objects.

## Local Cluster Note

A basic local `kind` cluster usually does not come with OIDC login configured. That is fine for this foundation. The key idea is to understand where human identity should come from in real clusters.

For local exercises, your kubeconfig may use a certificate-based admin identity. In production or shared clusters, prefer external identity with short-lived credentials and RBAC mapped to groups.
