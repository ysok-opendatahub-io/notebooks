# Pitch kit — Hermetic notebook images article

Companion notes for sharing [hermetic.md](./hermetic.md). Not part of the public article body.

## Short abstract (~150 characters)

Hermetic ODH/OpenShift AI notebook images: lockfile-pin RPM/pip/npm, prefetch with Cachi2, build offline on Konflux and GitHub Actions.

## Longer abstract

We built Open Data Hub and OpenShift AI workbench/runtime images so every dependency is lockfile-pinned and prefetched before a network-isolated build. The same Dockerfiles run locally, in GitHub Actions, and on Konflux. The article covers ODH vs subscribed RHDS/AIPCC variants, generating RPM and Python lockfiles, avoiding generic prefetch for Conforma SBOMs, and hard cases (codeserver npm, AIPCC ripgrep/pandoc wheels).

## Suggested pitch email

**Subject:** Pitch: Building hermetic notebook images for ODH / OpenShift AI

Hi [Editor / Advocate name],

I’d like to propose a Red Hat Developer article on how the notebooks team builds Open Data Hub and OpenShift AI workbench images hermetically.

**One-liner:** Lockfile-pin RPM/pip/npm, prefetch with Cachi2/Hermeto, build offline on Konflux and GHA — including codeserver’s npm graph and AIPCC-packaged ripgrep/pandoc.

**Why it fits:** Practical DevSecOps / containers / disconnected-build content aligned with Konflux, Conforma, and OpenShift AI. ~1,100 words with public GitHub links; no customer-confidential material.

**Draft:** https://github.com/ysok-opendatahub-io/notebooks/blob/article/hermetic-notebook-images/docs/articles/hermetic.md

**Tags / topics (suggested):** Containers, CI/CD, Python, DevSecOps, Disconnected environments, Red Hat OpenShift AI, Konflux

Happy to revise tone/length with editorial. Thanks!

## Forum / Slack teaser

We’re sharing how ODH / OpenShift AI notebook images are built **hermetically** (no network during `buildah`): committed RPM + pip + npm lockfiles, Cachi2/Hermeto prefetch on Konflux and GitHub Actions, and patterns that avoid opaque “generic” downloads so Conforma SBOMs stay useful. Deep dive includes codeserver (npm + patched ripgrep via AIPCC wheels) and pandoc-rhai. Full write-up: [link]. Feedback welcome.
