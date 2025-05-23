# bpftools‑arm64 automated builder

This repository is a **tiny wrapper** around  
[facebookexperimental/ExtendedAndroidTools](https://github.com/facebookexperimental/ExtendedAndroidTools).  
It builds **bpftools** for **Android ARM64 (aarch64)** and publishes the resulting archive.
[@see my issue "Feature request: publish pre-built Android binaries as GitHub **Releases**"](https://github.com/facebookexperimental/ExtendedAndroidTools/issues/116)
---

## Why?

* **Zero‑maintenance**: press a button or push a tag and get a fresh `bpftools-arm64.tar.gz`.
* **Free CI minutes**: public projects enjoy unlimited GitHub Actions time.
* **Reproducible**: Docker image + pinned upstream commit guarantee the same tool‑chain every run.

---

## Current status

* **GitHub Actions workflow** – automatically builds `bpftools-arm64.tar.gz`, publishes it as a Release asset **and** keeps a zipped artefact for debugging.
* **Manual steps (below)** – tested and working; useful when you need a local build or custom tweaks.

* **GitHub Actions workflow** – aims to build the archive automatically.
* **Manual steps (below)** – **tested and working**; follow them to obtain `bpftools-arm64.tar.gz`.

## Manual steps

The archive is generated as **`bpftools-arm64.tar.gz`** in the project root.

---

### Commands that **currently fail**

@see my issue **“make bpftools build fails: ld.lld: error: unable to find library ‑lLLVM when linking bpftrace‑aotrt”**  
<https://github.com/facebookexperimental/ExtendedAndroidTools/issues/115>

```bash
make bpftools      THREADS=$(nproc)   # fails
make bpftools-min  THREADS=$(nproc)   # fails
```

---

### A · Full build with monolithic **libLLVM.so** – **WORKS**

```bash
git clone https://github.com/facebookexperimental/ExtendedAndroidTools.git
cd ExtendedAndroidTools

# Build the Docker image (once)
./scripts/build-docker-image.sh

# Enter the build environment
./scripts/run-docker-build-env.sh

# Clean previous artefacts (optional)
make clean

# Enable strict shell and tracing
set -e
set -x

# Re‑build LLVM + Clang for the host (creates libLLVM.so)
make llvm HOST_ONLY=1   LLVM_EXTRA_CMAKE_FLAGS='-DLLVM_BUILD_LLVM_DYLIB=ON -DLLVM_LINK_LLVM_DYLIB=ON -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra"'   THREADS=$(nproc)

# Build bpftools for Android ARM64
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)
# or
make bpftools-min NDK_ARCH=arm64 THREADS=$(nproc) 
```

---

### B · “Slim” build – BCC tools only (skip AOT) – **NOT TESTED**

```bash
export SKIP_AOT=1
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)
```

---

### Verify the manual build

```bash
tar -tf bpftools-arm64.tar.gz | head
```
Output:
```
bpftools/
bpftools/bin/
bpftools/bin/bpftrace
bpftools/bin/bpftrace-aotrt
bpftools/bin/python3
bpftools/bin/python3-config
bpftools/bin/python3.10
bpftools/bin/python3.10-config
bpftools/bin/xzcat
bpftools/lib/
```
---

## What the workflow does

1. Check out this wrapper repository.  
2. Clone `facebookexperimental/ExtendedAndroidTools` (shallow).  
3. Build the project’s Docker image.  
4. Compile **bpftools** inside the container for `NDK_ARCH=arm64`.  
5. Upload `bpftools-arm64.tar.gz` as  
   * a workflow artefact (always), and  
   * a Release asset (on tag / manual dispatch).  
6. *(Optional)* Sign the tarball with Sigstore Cosign if `COSIGN_KEY` is provided.

---

## How to trigger a build

```bash
git tag v0.0.1
git push origin v0.0.1
```

A GitHub Actions run starts and attaches `bpftools-arm64.tar.gz` to the corresponding Release page.