# Building Hermetic Notebook Images for Open Data Hub and OpenShift AI

*How we made workbench and runtime image builds offline, reproducible, and release-ready, and what other teams can reuse.*

> Based on work in [opendatahub-io/notebooks](https://github.com/opendatahub-io/notebooks) and the downstream [red-hat-data-services/notebooks](https://github.com/red-hat-data-services/notebooks) fork.

---

If you have ever watched a container build fail because a package mirror hiccuped, or wondered whether last month’s image really matches what you ship today, you already know the pain hermetic builds are meant to solve.

For Open Data Hub (ODH) and Red Hat OpenShift AI notebook images, we moved to a simple rule: **nothing downloads during the image build**. Dependencies are pinned in lockfiles, prefetched ahead of time, and installed from a local cache while the build runs with no network. The same Dockerfiles work on a laptop, in GitHub Actions, and on Konflux.

This post is the story of that shift: why it mattered, what surprised us, and where to look if you want to reuse the pattern.

---

## Why “no network during build” matters

A hermetic container build is offline on purpose. Tools like [Cachi2](https://github.com/containerbuildsystem/cachi2) and [Hermeto](https://github.com/hermetoproject/hermeto) download every RPM, Python wheel, npm package, and Go module *before* `podman` or `buildah` starts. The Dockerfile only installs from that cache.

That buys you three things:

1. **Reproducibility:** the same lockfiles produce the same dependency tree, even when mirrors drift or floating tags move.
2. **Auditability:** packages are pinned by URL and checksum. SBOMs can name real ecosystems (`rpm`, `pip`, `npm`, `gomod`) instead of opaque tarball URLs.
3. **Compliance:** OpenShift AI product builds on Konflux require network isolation and [Conforma](https://conforma.dev/) checks. Upstream ODH images use the same hermetic pipeline; the stricter policy applies on the product path.

In short: hermetic is less a CI checkbox and more a packaging discipline: pin, prefetch, install offline.

For the full architecture diagram and environment matrix, see the in-repo [hermetic guide](https://github.com/opendatahub-io/notebooks/blob/main/docs/hermetic-guide.md).

---

## One Dockerfile, three places to build

The mental model is small:

**Commit lockfiles → prefetch into a cache → build with the network off.**

Locally and in GitHub Actions we run a shared prefetch script. On Konflux, a Tekton task does the same job. The Dockerfile does not care which environment filled the cache; it only installs from it.

Upstream (ODH) and downstream (RHDS / AIPCC) keep **separate** lock trees. Mixing them fails in subtle ways, for example CentOS versus RHEL FIPS package names. If you work with subscribed RHEL bases, start with [subscribed builds](https://github.com/opendatahub-io/notebooks/blob/main/docs/subscribed-builds.md).

Most Jupyter and runtime images share a repo-root prefetch tree. Codeserver keeps its own, because its dependency set (especially npm) is a different beast.

---

## The design rule that paid off

Early on we were tempted to “just download the binary” for awkward tools: ripgrep releases, pandoc tarballs, `oc` mirrors, VS Code marketplace extensions. That works until Conforma asks what those blobs actually *are*.

At Red Hat, the bar is higher than “it builds offline.” Product images must **build from source** or consume **approved dependencies**, such as Python wheels published through AIPCC. A GitHub release tarball prefetched as `type: generic` may satisfy hermetic networking, but it does not satisfy that packaging expectation.

The rule we settled on:

> Prefer typed ecosystems (`rpm`, `pip`, `npm`, `gomod`, or vendored source) over generic URL downloads. Avoid generic prefetch as much as possible.

Whenever we could turn a loose tarball into a first-class package (or an RPM from a known repo), SBOMs got clearer and release checks got quieter. Generic prefetch stayed mainly for things like GPG keys needed to verify prefetched RPMs, not as a dumping ground for binaries. If you truly cannot package a dependency that way, work with ProdSec on an exception. Treat exceptions as temporary bridges, not the default path.

How we generate RPM and Python lockfiles, and how to run prefetch yourself, lives in the [lockfile generators README](https://github.com/opendatahub-io/notebooks/blob/main/scripts/lockfile-generators/README.md).

---

## The hard cases: codeserver, ripgrep, and pandoc

**Codeserver** was the extreme end of the spectrum: a pinned [code-server](https://github.com/coder/code-server) submodule, a large npm graph, URL rewriting into the local cache, then `npm ci --offline`. If you only read one deep dive, make it the [codeserver README](https://github.com/opendatahub-io/notebooks/blob/main/codeserver/ubi9-python-3.12/README.md).

**Ripgrep** was a classic Node trap. Upstream `@vscode/ripgrep` downloads a binary in `postinstall`: fine on the open internet, fatal in a hermetic build. Working with AIPCC, we published a multi-arch `ripgrep` wheel, prefetched it as pip, pointed the environment at the installed binary, and patched the cached postinstall to copy that binary instead of downloading. The same idea generalizes to a lot of “downloads at install time” Node packages.

**Pandoc** followed the same path: swap a static tarball for a `pandoc-rhai` pip wheel so the SBOM sees a real Python package. For FIPS / payload-check context, see [fips.md](https://github.com/opendatahub-io/notebooks/blob/main/docs/fips.md).

Partner early on awkward binaries. Having multi-arch wheels *before* you need them in CI saves weeks of “works on my laptop, fails hermetically.”

---

## Passing Conforma without treating exceptions as the plan

On the OpenShift AI path, Conforma looks for hermetic trusted tasks, RPM signatures, multi-arch consistency, SBOM shape, and required labels. Policy lives in places like [rhtap-ec-policy](https://github.com/release-engineering/rhtap-ec-policy) and [conforma/policy](https://github.com/conforma/policy/tree/main/policy/release).

Exceptions (including ones coordinated with ProdSec) can bridge a gap. They should not replace fixing packaging shape: build from source or use approved dependencies, and keep generic prefetch rare. Typed ecosystems beat generic tarballs every time. How we validate locally is in [docs/conforma.md](https://github.com/opendatahub-io/notebooks/blob/main/docs/conforma.md).

---

## What we would tell another team

1. Treat hermetic as a **packaging practice**, not a single pipeline flag.
2. Split upstream and subscribed lockfiles when bases differ; never mix CentOS RPMs onto RHEL / AIPCC bases.
3. Build from source or use approved dependencies (for example AIPCC Python wheels). Avoid generic prefetch as much as possible; work with ProdSec if you need an exception.
4. Partner early for awkward binaries so multi-arch approved wheels exist before CI needs them.
5. Patch upstream installers that download at postinstall time. The codeserver ripgrep pattern travels well.
6. Automate lock renewal so “hermetic” does not mean “frozen forever.”
7. Budget CPU and memory for npm-heavy images; document where GitHub Actions and Konflux diverge.

---

## Where to go next

| If you want to… | Start here |
|---|---|
| Understand the architecture | [hermetic-guide.md](https://github.com/opendatahub-io/notebooks/blob/main/docs/hermetic-guide.md) |
| Generate lockfiles / run prefetch | [lockfile-generators README](https://github.com/opendatahub-io/notebooks/blob/main/scripts/lockfile-generators/README.md) |
| Copy a simple PipelineRun | [runtime-minimal PR pipeline](https://github.com/opendatahub-io/notebooks/blob/main/.tekton/odh-pipeline-runtime-minimal-cpu-py312-ubi9-pull-request.yaml) |
| Study the complex npm case | [codeserver README](https://github.com/opendatahub-io/notebooks/blob/main/codeserver/ubi9-python-3.12/README.md) and [codeserver PR pipeline](https://github.com/opendatahub-io/notebooks/blob/main/.tekton/odh-workbench-codeserver-datascience-cpu-py312-ubi9-pull-request.yaml) |
| Work with subscribed / AIPCC bases | [subscribed-builds.md](https://github.com/opendatahub-io/notebooks/blob/main/docs/subscribed-builds.md) |
| Validate Conforma locally | [conforma.md](https://github.com/opendatahub-io/notebooks/blob/main/docs/conforma.md) |

Questions and improvements are welcome via issues and PRs on [opendatahub-io/notebooks](https://github.com/opendatahub-io/notebooks). The scripts and docs above are meant to be reused beyond our workbench set. If you are wrestling with air-gapped or Conforma requirements on Konflux or OpenShift AI images, we hope this gives you a head start.
