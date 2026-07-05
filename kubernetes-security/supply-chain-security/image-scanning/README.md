---
foundation:
  topic: Kubernetes Security
  group: Supply Chain Security
  slug: image-scanning
  image: ../assets/supply-chain-security.svg
  order: 81
  title: "Image Scanning"
  description: "Finding known vulnerabilities in container images before and after deployment."
  publish: true
---

# Image Scanning

An image scanner reads the layers of a container image, inventories the OS packages and language dependencies inside, and matches them against vulnerability databases. The output is a list of known CVEs with severities and, usually, the version that fixes each one.

Scanning does not find novel attacks or backdoors. It answers one specific question well: **does this image contain components with known vulnerabilities?**

## Install Trivy

[Trivy](https://github.com/aquasecurity/trivy) is a fast, widely used open source scanner:

```bash
brew install trivy
```

Or run it as a container without installing anything:

```bash
docker run --rm aquasec/trivy:latest image nginx:1.25
```

## Scan An Image

```bash
trivy image nginx:1.25
```

Representative output:

```text
nginx:1.25 (debian 12.1)

Total: 147 (UNKNOWN: 1, LOW: 84, MEDIUM: 38, HIGH: 21, CRITICAL: 3)

┌────────────┬────────────────┬──────────┬─────────────────┬───────────────┐
│  Library   │ Vulnerability  │ Severity │ Installed Ver.  │  Fixed Ver.   │
├────────────┼────────────────┼──────────┼─────────────────┼───────────────┤
│ libssl3    │ CVE-2023-5678  │ HIGH     │ 3.0.9-1         │ 3.0.11-1      │
│ zlib1g     │ CVE-2023-45853 │ CRITICAL │ 1:1.2.13.dfsg-1 │               │
└────────────┴────────────────┴──────────┴─────────────────┴───────────────┘
```

A hundred-plus findings on a mainstream image is normal. Most come from the OS base layer, not the application.

## Gate The Pipeline

In CI, filter to the severities you act on and fail the build when they appear:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:1.0
```

Expected behavior: exit code `0` when clean, `1` when any HIGH or CRITICAL finding exists. That is what makes it a pipeline gate rather than a report.

Add `--ignore-unfixed` to skip findings that have no fixed version yet; you cannot patch your way out of those by rebuilding.

## Scanning Is Point-In-Time

A clean scan on Tuesday says nothing about Thursday: new CVEs are published against *existing* versions daily. The image did not change; the knowledge about it did. This has two consequences:

1. **Rescan what you run**, not just what you build. Schedule scans of deployed images, or use your registry's built-in scanning (Harbor, ECR, GCR, and others rescan continuously).
2. **Rebuild regularly.** A weekly rebuild on a patched base image clears most findings without any code change.

## The Biggest Lever Is The Base Image

The fastest way to fewer CVEs is shipping fewer packages:

| Base | Typical contents | Scan surface |
| --- | --- | --- |
| `debian`, `ubuntu` | Full OS userland | Large |
| `alpine` | Minimal musl-based userland | Small |
| Distroless / `scratch` | Runtime libraries only, no shell or package manager | Minimal |

Moving a Go or Java service from a full Debian base to a distroless image routinely removes the majority of findings, and removes the shell an attacker would use post-compromise.

## Triage Honestly

Not every CVE in an image is exploitable in your context: the vulnerable function may never be called, or the package may be present but unused. Suppress consciously with an ignore file:

```text
# .trivyignore, triaged 2026-07, re-review quarterly
# vulnerable code path not reachable: we do not use zlib's minizip
CVE-2023-45853
```

An ignore file with reasons and dates is triage. An ignore file without them is just hiding findings.

## Clean Up

Nothing to clean up, because scanning is read-only. Remove pulled test images if you want the disk back:

```bash
docker rmi nginx:1.25
```

## Practical Guidance

1. Gate CI on HIGH and CRITICAL with `--exit-code 1`; report the rest without blocking.
2. Rescan running images continuously (registry scanning or a scheduled job) because the vulnerability database moves even when images do not.
3. Rebuild images on a schedule so base image patches actually reach production.
4. Prefer distroless or minimal bases; fewer packages beats faster triage.
5. Keep `.trivyignore` entries commented with a reason and a review date.
6. Scanning finds known CVEs only. Pair it with signing and provenance, which defend against tampering that no CVE database will ever list.
