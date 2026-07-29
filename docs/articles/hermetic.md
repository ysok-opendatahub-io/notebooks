# Building Hermetic Notebook Images for Open Data Hub and OpenShift AI

*How we made workbench and runtime image builds offline, reproducible, and release-policy ready — and what other teams can reuse.*

> **Draft** for Red Hat Developer / internal sharing. Based on [opendatahub-io/notebooks](https://github.com/opendatahub-io/notebooks) (`development`) and the downstream [red-hat-data-services/notebooks](https://github.com/red-hat-data-services/notebooks) fork.

---

## Abstract

Open Data Hub and Red Hat OpenShift AI notebook images now build hermetically: dependencies are lockfile-pinned and prefetched before a network-isolated container build. The same Dockerfiles work locally, in GitHub Actions, and on Konflux, while satisfying Conforma-style release checks.

---

## Why hermetic builds matter

A hermetic container build runs **with no network access**. Every RPM, npm package, Python wheel, and Go module is downloaded and checksum-pinned *before* `podman` or `buildah` starts. The Dockerfile then installs only from that local cache ([Cachi2](https://github.com/containerbuildsystem/cachi2) / [Hermeto](https://github.com/hermetoproject/hermeto)).

That matters for three reasons:

1. **Reproducibility** — identical lockfiles produce identical dependency trees, regardless of mirror drift or floating tags.
2. **Auditability** — packages are pinned by URL and SHA-256. SBOMs can classify them by ecosystem (`rpm`, `pip`, `npm`, `gomod`) instead of opaque tarball URLs.
3. **Compliance** — Konflux product builds for OpenShift AI require network isolation (`hermetic: true`) and [Conforma](https://conforma.dev/) checks (hermetic task, SBOM, RPM signatures, required labels). Upstream Open Data Hub images use the same hermetic pipeline; Conforma enforcement applies on the product path (`quay.io/rhoai/`).

In-tree guides worth bookmarking:

- [hermetic-guide.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/hermetic-guide.md)
- [lockfile-generators README](https://github.com/opendatahub-io/notebooks/blob/development/scripts/lockfile-generators/README.md)
- [conforma.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/conforma.md)

---

## One Dockerfile, three environments

```
Committed lockfiles                 Prefetch                          Build (offline)
───────────────────                 ────────                          ───────────────
rpms.lock.yaml / artifacts.lock  →  Konflux: prefetch-dependencies →  Dockerfile.konflux.*
requirements.<flavor>.txt           Local/GHA: prefetch-all.sh          dnf / npm ci --offline
package-lock.json / go.mod             → cachi2/output/deps/...          uv --no-index
```

| Environment | Prefetch | Notes |
|---|---|---|
| Local | `scripts/lockfile-generators/prefetch-all.sh` | Makefile mounts `cachi2/output` and Hermeto RPM repos |
| GitHub Actions | Same script in `build-notebooks-TEMPLATE.yaml` | `--rhds` when building subscribed RHEL / AIPCC bases |
| Konflux | Tekton `prefetch-dependencies` | `hermetic: "true"` plus typed `prefetch-input` in `.tekton/*.yaml` |

Upstream (ODH) and downstream (RHDS / AIPCC) keep **separate** lock trees under `prefetch-input/odh/` and `prefetch-input/rhds/`:

| | ODH | RHDS / AIPCC |
|---|---|---|
| Base images | CentOS Stream / ODH bases | `quay.io/aipcc/base-images/...` |
| Prefetch | `prefetch-input/odh/` | `prefetch-input/rhds/` |
| Subscription | None | Required (RHEL CDN) |
| Conforma | Not applied | Applied on `quay.io/rhoai/` |

Mixing variants fails at install time — for example CentOS versus RHEL FIPS provider package names. See [subscribed-builds.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/subscribed-builds.md).

Jupyter and runtime images share a repo-root `prefetch-input/`. Codeserver keeps its own tree under `codeserver/ubi9-python-3.12/prefetch-input/` because its dependency set (especially npm) is different.

---

## Step 1: Generate lockfiles

### RPM lockfiles

Declare packages in `rpms.in.yaml`, resolve them with `rpm-lockfile-prototype` via [`create-rpm-lockfile.sh`](https://github.com/opendatahub-io/notebooks/blob/development/scripts/lockfile-generators/create-rpm-lockfile.sh), and commit `rpms.lock.yaml`.

Codeserver enables `moduleEnable: [nodejs:22]` so Hermeto includes module metadata for hermetic `dnf module enable`. Release kickoff regenerates ODH and RHDS locks for both the shared root tree and codeserver (`make kickoff-release`).

### Python requirements

```
pyproject.toml → pylock.<flavor>.toml → requirements.<flavor>.txt (hashed)
```

Flavors (`cpu` / `cuda` / `rocm`) match Dockerfile suffixes. Prefetch must set `RELEASE_PYTHON_VERSION=3.12` so environment markers match the image. Install offline with:

```bash
uv pip install --no-index --find-links /cachi2/output/deps/pip
```

Multi-arch wheels from the RHOAI package index avoid compiling from source inside hermetic builds.

### Prefer typed ecosystems over generic prefetch

The design rule that unlocked Conforma-friendly SBOMs:

> Prefer `rpm` / `pip` / `npm` / `gomod` (or vendored source) over `type: generic` URL downloads.

| Tempting generic download | What we do instead |
|---|---|
| ripgrep GitHub release | RHOAI `ripgrep` **pip** wheel + patched `@vscode/ripgrep` postinstall |
| pandoc static tarball | `pandoc-rhai` **pip** wheel (packaged with AIPCC) |
| `oc` mirror tarball | `openshift-clients` **RPM** |
| GitHub / codeload npm refs | Registry-only pins + patched lockfiles |
| mongocli binary fetch | Submodule + `type: gomod` |
| Marketplace `.vsix` at build | Repo `utils/*.vsix` via Git LFS |

Generic prefetch remains only where it belongs — mainly CentOS and EPEL **GPG keys** so prefetched RPMs can be verified — not as a dumping ground for binaries.

---

## Step 2: Prefetch, then build offline

**Konflux** PipelineRuns declare typed inputs, for example:

```yaml
- name: hermetic
  value: "true"
- name: prefetch-input
  value:
  - path: prefetch-input/odh   # use rhds downstream
    type: rpm
  - path: prefetch-input/odh
    type: generic
  - path: jupyter/minimal/ubi9-python-3.12
    type: pip
    binary:
      arch: x86_64,aarch64,ppc64le,s390x
    requirements_files: [requirements.cpu.txt]
```

**Local and GitHub Actions** run `prefetch-all.sh` (generic → pip → npm → rpm → gomod), then the Makefile mounts `cachi2/output`. Codeserver builds on GHA also pass `GHA_BUILD=true` (lower VS Code parallelism) and `--layers=false` to fit runner disk and RAM.

Validation is structural: Konflux runs the build with network isolation; locally you install only from `/cachi2/output/deps/{rpm,npm,pip,...}` via `dnf`, `npm ci --offline`, and `uv --no-index`.

---

## Hard cases: codeserver, ripgrep, and pandoc

**Codeserver** is the extreme case: [coder/code-server](https://github.com/coder/code-server) as a pinned submodule, dozens of npm lockfile paths in Tekton, URL rewriting to `file:///cachi2/...`, then `npm ci --offline`. Full walkthrough: [codeserver README](https://github.com/opendatahub-io/notebooks/blob/development/codeserver/ubi9-python-3.12/README.md).

Upstream `@vscode/ripgrep` downloads a binary in `postinstall.js` — incompatible with hermetic builds. Working with AIPCC, we:

1. Depend on the RHOAI-published `ripgrep` wheel in `pyproject.toml`.
2. Prefetch it as **pip**.
3. Install offline and set `RIPGREP_BINARY_PATH`.
4. Patch the cached `@vscode/ripgrep` postinstall to copy that binary instead of downloading.

**Pandoc** followed the same pattern: replace a generic static tarball with the `pandoc-rhai` pip wheel so Cachi2 and SBOMs treat it as a first-class Python package. See [fips.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/fips.md) for the FIPS / `check-payload` context.

---

## Passing Conforma on the product path

For OpenShift AI images, Conforma checks include hermetic and trusted tasks, RPM signatures, multi-arch RPM version consistency, SBOM attributes, and required labels.

- Policy data: [rhtap-ec-policy](https://github.com/release-engineering/rhtap-ec-policy)
- Checks: [conforma/policy/release](https://github.com/conforma/policy/tree/main/policy/release)
- Product exceptions: release-engineering `konflux-release-data` Enterprise Contract policy for `registry-rhoai-{stage,prod}`

Exceptions are a temporary bridge — not a substitute for fixing packaging shape. Typed ecosystems beat generic tarballs every time. Local validation is documented in [docs/conforma.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/conforma.md).

---

## Lessons learned

1. Hermetic is a **packaging discipline**, not a single CI flag: pin, prefetch, install offline.
2. Split upstream and subscribed lockfiles when bases differ; never mix CentOS RPMs onto RHEL / AIPCC bases.
3. Avoid generic prefetch for binaries — push work into rpm / pip / npm / gomod (or vendor source).
4. Partner early for awkward binaries (ripgrep, pandoc) so multi-arch wheels exist before you need them in CI.
5. Patch upstream installers that download at postinstall time; the codeserver ripgrep pattern generalizes to many Node ecosystems.
6. Automate lock renewal so “hermetic” does not mean “frozen forever.”
7. Budget CPU and memory for npm-heavy images; document GHA versus Konflux differences.

---

## Call to action

If you build Konflux or OpenShift AI images and are wrestling with air-gapped or Conforma requirements, start here:

| Goal | Link |
|---|---|
| Architecture | [docs/hermetic-guide.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/hermetic-guide.md) |
| Run prefetch locally | [scripts/lockfile-generators/README.md](https://github.com/opendatahub-io/notebooks/blob/development/scripts/lockfile-generators/README.md) |
| Simple PipelineRun | [odh-pipeline-runtime-minimal…](https://github.com/opendatahub-io/notebooks/blob/development/.tekton/odh-pipeline-runtime-minimal-cpu-py312-ubi9-pull-request.yaml) |
| Complex (npm) case | codeserver `.tekton/odh-workbench-codeserver-*-pull-request.yaml` |
| Subscription / AIPCC bases | [docs/subscribed-builds.md](https://github.com/opendatahub-io/notebooks/blob/development/docs/subscribed-builds.md) |

Questions and improvements are welcome via issues and PRs on [opendatahub-io/notebooks](https://github.com/opendatahub-io/notebooks). The scripts and docs above are meant to be reused beyond our workbench set.
