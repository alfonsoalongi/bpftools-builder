## Fix dockerfile
Il Makefile però, per storicità interna a Meta, cerca sempre:
/opt/ndk/android-ndk-r27b/build/cmake/android.toolchain.cmake
    ecco da dove nasce l’errore “Could not find toolchain file”.

RUN  /jammy-install-deps.sh && \
     /download-ndk.sh /opt/ndk   && \
     ln -s /opt/ndk/android-ndk-r26b /opt/ndk/android-ndk-r27b

## A · Build completo con libLLVM monolitica

```bash
git clone https://github.com/facebookexperimental/ExtendedAndroidTools.git
cd ExtendedAndroidTools

# Crea l’immagine (una sola volta)
./scripts/build-docker-image.sh

# 1. Avvia l’ambiente
./scripts/run-docker-build-env.sh

# 2 Pulisce l''ambiente (opzionale)
make clean

# 3. Attiva gli errori e trace
set -e;
set -x;

# 4. Compila solo l’LLVM host (dylib monolitica + Clang)
make llvm HOST_ONLY=1 \
  LLVM_EXTRA_CMAKE_FLAGS='-DLLVM_BUILD_LLVM_DYLIB=ON -DLLVM_LINK_LLVM_DYLIB=ON -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra"' \
  THREADS=$(nproc)

# 5. Compila bpftools per arm64
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)

# 6 Il tarball finale è in out/archives/bpftools-arm64.tar.gz
exit
```


---

## B · Build “slim” solo bcc-tools (skip AOT)

# Compila tutto in un colpo (niente rebuild LLVM)

export SKIP_AOT=1
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)


out/archives/bpftools-arm64.tar.gz



## Copiare sul device Android

```bash
adb push out/bpftools*-arm64.tar.gz /data/local/tmp
adb shell "cd /data/local/tmp && tar xf bpftools*-arm64.tar.gz"
adb shell "cd /data/local/tmp && LD_LIBRARY_PATH=. ./tcpconnect -h"
```

Per un modulo Magisk basta spostare `tcpconnect*` e le rispettive
`libbcc.so`, `libbpf.so`, `libc++_shared.so` in
`system/bin/` e `system/lib64/` del tuo zip.

---
