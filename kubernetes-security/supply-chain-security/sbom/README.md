---
foundation:
  topic: Kubernetes Security
  group: Supply Chain Security
  slug: sbom
  image: ../assets/supply-chain-security.svg
  order: 82
  title: "SBOM"
  description: "Generating a software bill of materials to know exactly what is inside an image."
  publish: true
---

# SBOM

A software bill of materials (SBOM) is a machine-readable inventory of every component in an artifact: OS packages, language dependencies, versions, and licenses. It is the ingredients label for a container image.

The motivating scenario is incident response. When Log4Shell landed, every security team on earth asked the same question: *where are we running log4j?* Teams with SBOMs queried a database and had an answer in minutes. Teams without them started pulling and scanning every image they had ever deployed.

## Two Formats

Two standards dominate; every serious tool speaks both:

| Format | Origin | Notes |
| --- | --- | --- |
| SPDX | Linux Foundation | ISO standard, strong license tooling |
| CycloneDX | OWASP | Security-focused, rich vulnerability linkage |

Pick one and stay consistent. The choice matters less than actually generating and keeping them.

## Generate An SBOM

[Syft](https://github.com/anchore/syft) generates SBOMs from images, directories, and archives:

```bash
brew install syft
syft nginx:1.25 -o spdx-json > nginx-1.25.spdx.json
```

Expected output:

```text
 ✔ Loaded image            nginx:1.25
 ✔ Parsed image
 ✔ Cataloged contents
   ├── ✔ Packages           [148 packages]
   └── ✔ File digests       [3,285 files]
```

The JSON file now lists every package with name, version, and origin: greppable, diffable, and queryable:

```bash
jq '.packages[] | select(.name | contains("ssl")) | {name, versionInfo}' nginx-1.25.spdx.json
```

```text
{
  "name": "libssl3",
  "versionInfo": "3.0.9-1"
}
```

## Scan The SBOM Instead Of The Image

An SBOM decouples *inventory* from *analysis*. [Grype](https://github.com/anchore/grype), Syft's companion scanner, matches an SBOM against the current vulnerability database without touching the image again:

```bash
grype sbom:nginx-1.25.spdx.json
```

This is what makes the next Log4Shell tractable: generate the SBOM once at build time, then re-run matching against fresh CVE data every day, with no image pulls and no rescans, and it works even for images whose registry is gone.

## Attach The SBOM To The Image

An SBOM in a folder gets lost. Attach it to the image in the registry as a signed attestation so it travels with the artifact:

```bash
cosign attest --predicate nginx-1.25.spdx.json --type spdxjson ttl.sh/demo-app:1h
```

Anyone who can pull the image can now retrieve and verify its inventory:

```bash
cosign verify-attestation --type spdxjson --key cosign.pub ttl.sh/demo-app:1h
```

Signing and attestation mechanics are covered on the next page.

## Know The Limits

- **SBOMs see what is visible.** A statically-linked binary with vendored dependencies may catalog as one opaque package. Language-aware builds (generating the SBOM from the lockfile at build time, not from the final image) produce better inventories.
- **Quality varies by ecosystem.** OS packages and npm/PyPI/Go modules catalog well; hand-copied vendored code does not.
- **An SBOM is a snapshot of a build.** Rebuild the image, regenerate the SBOM.

## Clean Up

```bash
rm nginx-1.25.spdx.json
docker rmi nginx:1.25
```

## Practical Guidance

1. Generate an SBOM for every image at build time, in the same CI job that builds it, because that is when the most information is available.
2. Store SBOMs centrally and attach them to images as attestations; both, not either.
3. Re-match stored SBOMs against fresh vulnerability data on a schedule; this is cheaper and faster than rescanning images.
4. Prefer generating from source and lockfiles over scanning the finished image when your build allows it.
5. Ask vendors for SBOMs with their software; SLSA and modern procurement language make this a normal request.
6. Treat "we cannot say what is in this image" as the finding, even when no CVE is currently known.
