# bpftools‑arm64 automated builder

This repository is a tiny **wrapper** around
[`facebookexperimental/ExtendedAndroidTools`](https://github.com/facebookexperimental/ExtendedAndroidTools)
that builds **bpftools** for the **Android ARM64 (aarch64)** architecture manually or with a workflow
and publishes the result as a GitHub Release asset.

## Why?

* **Zero‑maintenance** binary drops — press a button or push a tag and get
  `bpftools-arm64.tar.gz`.
* **Free**: public repositories have unlimited GitHub Actions minutes.
* **Reproducibility**: the Docker image + pinned upstream commit guarantee
  identical toolchains across runs.

## Manual steps

#  Output in: bpftools-arm64.tar.gz

# All the steps does not work
@see my issue "make bpftools build fails: ld.lld: error: unable to find library -lLLVM when linking bpftrace-aotrt" at  https://github.com/facebookexperimental/ExtendedAndroidTools/issues/115
make bpftools THREADS=$(nproc)
make bpftools-min THREADS=$(nproc)

## A · Build completo con libLLVM monolitica - IT WORKS

```bash
git clone https://github.com/facebookexperimental/ExtendedAndroidTools.git
cd ExtendedAndroidTools

# Crea l’immagine (una sola volta)
./scripts/build-docker-image.sh

# 1. Avvia l’ambiente
./scripts/run-docker-build-env.sh

# 2 Pulisce l''ambiente (opzionale)
make clean

# 3.Si ferma se trova un errore e attiva il tracing
set -e;
set -x;

# 4. Compila solo l’LLVM host (dylib monolitica + Clang)
make llvm HOST_ONLY=1 \
  LLVM_EXTRA_CMAKE_FLAGS='-DLLVM_BUILD_LLVM_DYLIB=ON -DLLVM_LINK_LLVM_DYLIB=ON -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra"' \
  THREADS=$(nproc)

# 5. Compila bpftools per arm64
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)


## B · Build “slim” solo bcc-tools (skip AOT) - NON PROVATA

# Compila tutto in un colpo (niente rebuild LLVM)

export SKIP_AOT=1
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)


# Verifica build manuale
# dovresti vedere: bpftools/bin/tcpconnect, tcpconnect6, … ecc.
tar -tf bpftools-arm64.tar.gz | head


## What the workflow does - Currently the workflow does not works!

1. Checks out this wrapper repository (tiny).
2. Clones `facebookexperimental/ExtendedAndroidTools` at depth 1.
3. Builds the project’s Docker image (`./scripts/build-docker-image.sh`).
4. Compiles **bpftools** inside the container for `NDK_ARCH=arm64`.
5. Uploads `out/archives/bpftools-arm64.tar.gz` as:
   * a workflow *artifact* (always), and
   * a Release asset (on tag / manual dispatch).
6. *(Optional)* Signs the tarball with [Sigstore Cosign] if the secret
   `COSIGN_KEY` is configured.

## How to trigger a build 

Create a tag
git tag v0.0.1
git push origin v0.0.1


