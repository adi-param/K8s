---
foundation:
  topic: Kubernetes Security
  group: Supply Chain Security
  slug: overview
  image: assets/supply-chain-security.svg
  order: 80
  title: "Overview"
  description: "Trusting what you run: the risks in the path from source code to a running container image."
  publish: true
---

# Overview

Everything so far hardened the cluster and the workloads inside it. Supply chain security asks an earlier question: **can you trust what you deployed in the first place?**

The supply chain is everything between source code and a running pod:

```text
source -> dependencies -> build -> image -> registry -> admission -> pod
```

Every arrow is an attack surface. A cluster with perfect RBAC and a `restricted` Pod Security Standard still runs whatever image the deployment references, and that image is the product of a long chain you mostly did not build yourself.

## How Supply Chains Get Attacked

These are not hypothetical; each pattern has real incidents behind it:

- **Compromised base images:** a popular image on a public registry is backdoored, and everything built on top inherits the implant.
- **Typosquatting and dependency confusion:** a malicious package named one character away from the real one, or a public package that shadows an internal name, gets pulled into your build.
- **Build system compromise:** the attacker modifies artifacts *during* the build, so source review finds nothing (the SolarWinds pattern).
- **Registry tampering:** a tag like `myapp:1.4` is mutable; whoever can push to the registry can silently replace what it points to.

The common thread: by the time the pod starts, the malicious code is inside an image you asked for.

## The Four Controls

This section covers four controls, each guarding a different stage:

| Control | Question it answers | Pipeline stage |
| --- | --- | --- |
| Image scanning | Does this image contain known vulnerabilities? | Build and registry |
| SBOM | What exactly is inside this image? | Build |
| Image signing | Who published this image, and is it unmodified? | Registry to cluster |
| Provenance | How, from what source, and by which system was it built? | Build to cluster |

They stack. Scanning finds known-bad components. An SBOM makes the component inventory queryable when the next big CVE drops. A signature proves the image came from your pipeline and was not swapped in the registry. Provenance proves the pipeline itself built it the way you think it did.

## SLSA As The Map

[SLSA](https://slsa.dev) (Supply-chain Levels for Software Artifacts) grades build integrity from level 0 (nothing) to level 3 (hardened, provenance-generating build platform). You do not need to chase a level to use it: SLSA is most useful as a map of which attacks each control stops, and as shared vocabulary when asking vendors how their artifacts are built.

## Admission Control Closes The Loop

None of these controls matter if the cluster runs unsigned, unscanned images anyway. The enforcement point is admission control from earlier in this foundation: policies that reject pods whose images are unsigned, come from untrusted registries, or lack required attestations. Build-side controls produce evidence; admission control demands it.

The next pages work through scanning, SBOMs, signing, and provenance hands-on.
