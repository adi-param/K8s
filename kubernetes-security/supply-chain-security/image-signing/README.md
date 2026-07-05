---
foundation:
  topic: Kubernetes Security
  group: Supply Chain Security
  slug: image-signing
  image: ../assets/supply-chain-security.svg
  order: 83
  title: "Image Signing"
  description: "Signing and verifying image provenance with cosign so only trusted images run."
  publish: true
---

# Image Signing

A container tag is a mutable pointer. `myapp:1.4` means "whatever the registry currently returns for that name", and anyone with push access, or control of the registry, can change it. Scanning cannot catch this: a backdoored image can scan perfectly clean.

A signature fixes the trust problem directly. Signing an image binds its exact digest to a key you control; verification proves two things at once:

- **origin:** the image was signed by your pipeline's key, and
- **integrity:** the content has not changed by a single byte since.

## Cosign

[Cosign](https://github.com/sigstore/cosign) from the Sigstore project is the de facto standard. It signs image digests and stores signatures in the registry next to the image, so no extra infrastructure is needed.

```bash
brew install cosign
```

Cosign supports two modes:

- **Key-based:** you generate and protect a key pair. Simple to reason about; you own key rotation and secrecy.
- **Keyless:** no long-lived key. Cosign gets a short-lived certificate from Fulcio tied to an OIDC identity (like a GitHub Actions workflow), and logs the signature in Rekor, a public transparency log. Verification checks identity, not a key file.

Keyless is the better fit for CI; key-based is easier to learn with, so the lab uses it.

## Sign An Image

Generate a key pair (choose any password when prompted):

```bash
cosign generate-key-pair
```

Expected output:

```text
Private key written to cosign.key
Public key written to cosign.pub
```

Push a test image to [ttl.sh](https://ttl.sh), an anonymous registry whose images expire, ideal for labs (`1h` in the tag is the time to live):

```bash
docker pull busybox:1.36
docker tag busybox:1.36 ttl.sh/adi-signing-demo:1h
docker push ttl.sh/adi-signing-demo:1h
```

Sign it:

```bash
cosign sign --key cosign.key ttl.sh/adi-signing-demo:1h
```

Cosign warns that the tag is mutable and signs the resolved digest. That warning is the whole point of this page.

## Verify

```bash
cosign verify --key cosign.pub ttl.sh/adi-signing-demo:1h
```

Expected output:

```text
Verification for ttl.sh/adi-signing-demo:1h --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
```

Verify an image nobody signed and the failure is unambiguous:

```bash
cosign verify --key cosign.pub ttl.sh/some-other-image:1h
```

```text
Error: no signatures found
```

## Enforce Signatures At Admission

Signatures are only useful if unsigned images cannot run. Kyverno (from the Admission Control section) verifies signatures in the admission path:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: require-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["lab"]
      verifyImages:
        - imageReferences:
            - "ttl.sh/adi-*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      <contents of cosign.pub>
                      -----END PUBLIC KEY-----
```

With this policy applied, a pod using an unsigned `ttl.sh/adi-*` image is rejected at admission, and signed images are additionally **pinned to their digest** by Kyverno, so the tag-mutation attack disappears entirely.

## Clean Up

```bash
kubectl delete clusterpolicy verify-image-signatures --ignore-not-found
rm cosign.key cosign.pub
docker rmi busybox:1.36 ttl.sh/adi-signing-demo:1h
```

ttl.sh images expire on their own.

## Practical Guidance

1. Sign in CI, immediately after push, in the same job that built the image.
2. Prefer keyless signing in CI so there is no private key to leak; verification then asserts "signed by this repo's release workflow" instead of "signed by whoever holds the key".
3. Protect key-based signing keys like production credentials: KMS-backed, never in the repo.
4. Enforce verification with admission policy scoped to your registries; a signature nobody checks is decoration.
5. Roll out verification in audit mode first, because unsigned base images and third-party charts will surface immediately.
6. Signing proves *who* published an image, not *how* it was built. The next page, provenance, covers the second half.
