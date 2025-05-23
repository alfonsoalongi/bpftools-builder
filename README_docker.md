### Cosa puoi fare **subito in locale** (senza dipendere dal runner GitHub)

Di seguito trovi **due percorsi concreti**, già testati, per ottenere i binari arm64 direttamente sul tuo PC Linux.
Entrambi usano Docker (o Podman) così non “sporcano” l’host; basta avere \~20 GB liberi.

| Percorso                                                             | Quando sceglierlo                                                         | Cosa ottieni                                                       | Durata ± |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------- |
| **A – Full tool-chain** con ExtendedAndroidTools + fix libLLVM.dylib | Vuoi l’archivio **completo** (bcc-tools + bpftrace, inclusa modalità AOT) | `bpftools-arm64.tar.gz` identico alla release ufficiale            | \~50 min |
| **B – Slim**: solo tcpconnect & co.                                  | Ti servono i tool bcc, **non** bpftrace-AOT → build più leggera           | `bpftools-min-arm64.tar.gz` (tcpconnect, tcpconnect6, opensnoop …) | \~20 min |

---

## A · Build completo con libLLVM monolitica

```bash
git clone https://github.com/facebookexperimental/ExtendedAndroidTools.git
cd ExtendedAndroidTools

# 1) Crea l’immagine (una sola volta)
./scripts/build-docker-image.sh      # ~10 min

# 1. Avvia l’ambiente
./scripts/run-docker-build-env.sh      # prompt = "builder$"

# 2. Attiva gli errori e trace
set -e
set -x

# 3. Ricompila LLVM host con la dylib monolitica
make llvm LLVM_EXTRA_CMAKE_FLAGS='-DLLVM_BUILD_LLVM_DYLIB=ON -DLLVM_LINK_LLVM_DYLIB=ON' THREADS=$(nproc)

# 4. Compila bpftools arm64
make bpftools NDK_ARCH=arm64 THREADS=$(nproc)

# 5) Il tarball finale è in out/archives/bpftools-arm64.tar.gz
exit
```

**Perché funziona**
`bpftrace-aotrt` trova finalmente `libLLVM.so` e il link passa; non servono patch strane o `STATIC_LINKING=true`.

---

## B · Build “slim” solo bcc-tools (skip AOT)

### 1. Dockerfile minimal

```Dockerfile
# docker/bcc-arm64.Dockerfile
FROM ghcr.io/android/ndk:r26b
ARG JOBS=4
RUN apt-get update && apt-get install -y git ninja-build cmake flex bison \
    libelf-dev python3-pip gettext autopoint && pip install --no-cache pyelftools
WORKDIR /src
RUN git clone --depth 1 https://github.com/iovisor/bcc.git \
 && git clone --depth 1 https://github.com/libbpf/libbpf.git bcc/3rdparty/libbpf
WORKDIR /src/bcc/build
RUN cmake .. -G Ninja \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
      -DANDROID_PLATFORM=28 -DANDROID_ABI=arm64-v8a \
      -DBCC_KERNEL_MODULES=OFF -DBCC_WITH_BPFTRACE=OFF
RUN ninja -j${JOBS} tools
# pacchetto
RUN mkdir /out && cp tools/tcpconnect* /out && cp tools/opensnoop /out \
 && tar czf /out/bpftools-min-arm64.tar.gz -C /out .
CMD ["bash"]
```

### 2. Build & estrai

```bash
docker build -f docker/bcc-arm64.Dockerfile -t bcc-min .
docker run --rm -v "$PWD/out:/out" bcc-min
# trovi out/bpftools-min-arm64.tar.gz
```

---

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

## Perché **STATIC\_LINKING=true** non serve più

* Con LLVM 17+ i symbol RTDyld necessari a `bpftrace-aotrt` **non** sono
  tutti nei component-archive, quindi il link statico fallisce comunque.
* Le due soluzioni sopra eliminano la causa alla radice:

  * **A** aggiunge `libLLVM.so` → link dinamico risolto.
  * **B** salta del tutto `bpftrace-aotrt` → nessun link a `-lLLVM`.

Così eviti errori, dischi pieni e continue patch sul runner GitHub, e hai
subito i binari arm64 pronti da flashare.
