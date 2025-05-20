# bpftools‑arm64 automated builder

This repository is a tiny **wrapper** around
[`facebookexperimental/ExtendedAndroidTools`](https://github.com/facebookexperimental/ExtendedAndroidTools)
that builds **bpftools** for the **Android ARM64 (aarch64)** architecture
and publishes the result as a GitHub Release asset.

## Why?

* **Zero‑maintenance** binary drops — press a button or push a tag and get
  `bpftools-arm64.tar.gz`.
* **Free**: public repositories have unlimited GitHub Actions minutes.
* **Reproducibility**: the Docker image + pinned upstream commit guarantee
  identical toolchains across runs.

## How to trigger a build


| Trigger                                         | What happens                                                            |
| ------------------------------------------------- | ------------------------------------------------------------------------- |
| **Manual** (`Run workflow` in the Actions tab) | Build & release using the*tag* you enter.                               |
| **Push a Git tag** like `v1.3.0`                | Build & release under that tag automatically.                           |
| **Pull request**                                | Builds the tarball and uploads it as an*artifact* (no release).         |
| **Weekly schedule** (Sunday 03:00 UTC)         | Rebuilds against the latest upstream and updates the release if needed. |

## What the workflow does

1. Checks out this wrapper repository (tiny).
2. Clones `facebookexperimental/ExtendedAndroidTools` at depth 1.
3. Builds the project’s Docker image (`./scripts/build-docker-image.sh`).
4. Compiles **bpftools** inside the container for `NDK_ARCH=arm64`.
5. Uploads `out/archives/bpftools-arm64.tar.gz` as:
   * a workflow *artifact* (always), and
   * a Release asset (on tag / manual dispatch).
6. *(Optional)* Signs the tarball with [Sigstore Cosign] if the secret
   `COSIGN_KEY` is configured.

## Manual steps

```bash
git clone https://github.com/facebookexperimental/ExtendedAndroidTools.git
cd ExtendedAndroidTools
# Build the Docker image
./scripts/build-docker-image.sh
# Run the environment
./scripts/run-docker-build-env.sh
# Create the artifact in out/archives/bpftools-arm64.tar.gz
make bpftools THREADS=$(nproc)
```

## Extra goodies

* **Docker‑layer caching** – uncomment the stub in the workflow to shave
  minutes off repeated builds.
  It uses `actions/cache` + Buildx’s `--cache-from/--cache-to`.
* **Weekly rebuilds** – already enabled; adjust the `cron:` as needed.
* **Pull‑request verification** – every PR gets a fresh artifact so you can
  test before merging or tagging.
* **Release signing (Cosign)** – add an encrypted private key as secret
  `COSIGN_KEY`, then enable the step to attach a detached signature.

> 💡 *Multi‑architecture builds* were intentionally left out — this wrapper
> focuses solely on the Android ARM64 target.

## Getting the binaries

Head to **Releases** → download `bpftools-arm64.tar.gz`, or grab the artifact
from the latest workflow run for quick, pre‑release testing.

---

© 2025. Released under the MIT License.
