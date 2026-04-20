# Xyce MPI install on landsman servers (user-space, no sudo)

Full reproducible procedure for building Xyce 7.11-dev with MPI, HB, FFT,
and ADMS plugin support on landsman-family servers (RHEL 8.10, no sudo,
conda-forge as the dependency satisfier). Derived from a 2026-04-14 install
on landsman5 that took ~2 h of wall time (mostly Trilinos source build).

**Every step below was empirically verified.** Paths assume working on
`/data/rzhao/` since `/home` is full on all landsman servers (see
`feedback_landsman5_workdir.md`). Substitute `/data/<user>/` for other users.

---

## Prerequisites checklist

- RHEL 8.x, gcc 8.5+ (system), cmake 3.22+ (system or miniforge), bash
- **No sudo required.** All installs are user-space.
- SSH key already authorized (no 2FA for landsman).
- ≥ 50 GB free on `/data/<user>`.

---

## Stage 0 — miniforge (~3 min)

```bash
cd /data/<user>
curl -fsSL -o /tmp/miniforge.sh \
  'https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh'
bash /tmp/miniforge.sh -b -p /data/<user>/miniforge
/data/<user>/miniforge/bin/conda --version
```

---

## Stage 1 — conda env with compiler + libs **(critical: NO trilinos)** (~5 min)

```bash
source /data/<user>/miniforge/etc/profile.d/conda.sh

mamba create -y -n xyce-deps -c conda-forge \
    suitesparse=7.10 \
    fftw=3.3.10 \
    openblas \
    bison=3.8 \
    flex \
    cmake \
    make \
    gxx_linux-64=13 \
    gfortran_linux-64=13 \
    mpich=4.3
```

**Critical — DO NOT install `trilinos` from conda-forge.** conda-forge's
Trilinos is built without Xyce's required `EpetraExt_BUILD_BTF=ON` feature.
If you install it, its `EpetraExt_config.h` will shadow your source-built
Trilinos's, and Xyce's CMake probe will silently fail with confusing
"HAVE_BTF not found" errors. We will build Trilinos from source.

If you accidentally install conda trilinos: `mamba remove -y trilinos` then
`mamba install -y mpich` (removing trilinos also pulls out mpich as a dep).

---

## Stage 2 — admsXml (~2 min)

Xyce's ADMS/Verilog-A plugin path needs admsXml ≥ 2.3. System RHEL 8
doesn't have it; build from Qucs/ADMS source:

```bash
cd /data/<user>
git clone --depth 1 https://github.com/Qucs/ADMS.git
cd ADMS
./bootstrap && ./configure --prefix=/data/<user>/adms_install
make -j8 && make install
/data/<user>/adms_install/bin/admsXml --version   # expect 2.3.7
```

---

## Stage 3 — Trilinos source build (~25–40 min)

**Do NOT use conda-forge Trilinos.** It lacks EpetraExt features Xyce
requires (BTF, EXPERIMENTAL, GRAPH_REORDERINGS) and pulls in a CUDA-built
Kokkos that fails on non-CUDA builds.

```bash
cd /data/<user>
curl -fsSL -o trilinos-16.1.zip \
  'https://github.com/trilinos/Trilinos/archive/refs/tags/trilinos-release-16-1-0.zip'
unzip -q trilinos-16.1.zip
mv Trilinos-trilinos-release-16-1-0 Trilinos

mkdir trilinos_build_mpi && cd trilinos_build_mpi
cat > cfg.sh << 'EOF'
#!/bin/bash
set -e
source /data/<user>/miniforge/etc/profile.d/conda.sh
conda activate xyce-deps
cmake \
  -C /data/<user>/xyce_src/cmake/trilinos/trilinos-base.cmake \
  -DCMAKE_INSTALL_PREFIX=/data/<user>/trilinos_install_mpi \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=$CONDA_PREFIX/bin/mpicc \
  -DCMAKE_CXX_COMPILER=$CONDA_PREFIX/bin/mpic++ \
  -DCMAKE_Fortran_COMPILER=$CONDA_PREFIX/bin/mpifort \
  -DTPL_ENABLE_MPI=ON \
  -DMPI_BASE_DIR=$CONDA_PREFIX \
  -DBLAS_LIBRARIES=$CONDA_PREFIX/lib/libopenblas.so \
  -DLAPACK_LIBRARIES=$CONDA_PREFIX/lib/libopenblas.so \
  -DTPL_AMD_INCLUDE_DIRS=$CONDA_PREFIX/include/suitesparse \
  -DTPL_AMD_LIBRARIES=$CONDA_PREFIX/lib/libamd.so \
  -DTrilinos_ENABLE_Isorropia=ON \
  -DEpetraExt_BUILD_BTF=ON \
  -DEpetraExt_BUILD_EXPERIMENTAL=ON \
  -DEpetraExt_BUILD_GRAPH_REORDERINGS=ON \
  -DTrilinos_ENABLE_KokkosKernels=ON \
  -DTrilinos_ENABLE_TESTS=OFF \
  /data/<user>/Trilinos
EOF
chmod +x cfg.sh

# Full build (persistent env via source-then-build):
nohup bash -c 'source /data/<user>/miniforge/etc/profile.d/conda.sh && \
               conda activate xyce-deps && \
               bash cfg.sh > configure.log 2>&1 && \
               make -j 16 install > install.log 2>&1 && \
               echo DONE > /tmp/trilinos_done.flag' \
      > /tmp/trilinos_wrapper.log 2>&1 &
```

**Key flags** (each learned from a failed build):
- `-DTPL_AMD_INCLUDE_DIRS=$CONDA_PREFIX/include/suitesparse` — conda-forge
  suitesparse puts headers in `/include/suitesparse/`, not `/include/`.
- `-DTrilinos_ENABLE_Isorropia=ON` — Xyce requires Isorropia for MPI
  builds; without it Xyce's CMake error messages are misleading (blame
  EpetraExt instead).
- `-DEpetraExt_BUILD_BTF=ON` + `BUILD_EXPERIMENTAL` + `BUILD_GRAPH_REORDERINGS`
  — explicit Xyce requirements; omit and Xyce configure halts.

Verify after install:
```bash
ar t /data/<user>/trilinos_install_mpi/lib64/libepetraext.a | grep -i btf
# Expect: EpetraExt_BTF_CrsGraph.cpp.o  EpetraExt_BTF_CrsMatrix.cpp.o ...
ls /data/<user>/trilinos_install_mpi/lib64/cmake/Isorropia/
# Expect: non-empty
grep HAVE_BTF /data/<user>/trilinos_install_mpi/include/EpetraExt_config.h
# Expect: #define HAVE_BTF 1
```

---

## Stage 4 — Xyce source build (~15 min)

```bash
cd /data/<user>
git clone --depth 1 https://github.com/Xyce/Xyce.git xyce_src

mkdir xyce_mpi_build && cd xyce_mpi_build

cat > build_wrapper.sh << 'EOF'
#!/bin/bash
set -e
# PERSIST conda env through make sub-shells — do NOT split sourcing
# between cfg.sh and make. See "env-scope bug" gotcha below.
source /data/<user>/miniforge/etc/profile.d/conda.sh
conda activate xyce-deps
export PATH=/data/<user>/adms_install/bin:$PATH
cd /data/<user>/xyce_mpi_build
cmake \
  -DTrilinos_DIR=/data/<user>/trilinos_install_mpi/lib64/cmake/Trilinos \
  -DCMAKE_INSTALL_PREFIX=/data/<user>/xyce_mpi_install \
  -DCMAKE_BUILD_TYPE=Release \
  -DXyce_PLUGIN_SUPPORT=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DCMAKE_C_COMPILER=$CONDA_PREFIX/bin/mpicc \
  -DCMAKE_CXX_COMPILER=$CONDA_PREFIX/bin/mpic++ \
  -DCMAKE_Fortran_COMPILER=$CONDA_PREFIX/bin/mpifort \
  -DCMAKE_CXX_FLAGS='-fopenmp' \
  -DADMS_EXECUTABLE=/data/<user>/adms_install/bin/admsXml \
  /data/<user>/xyce_src > configure.log 2>&1
make -j 16 install > install.log 2>&1
echo DONE > /tmp/xyce_mpi_done.flag
EOF
chmod +x build_wrapper.sh

nohup ./build_wrapper.sh > /tmp/xyce_wrapper.log 2>&1 &
```

**Key flags:**
- `-DXyce_PLUGIN_SUPPORT=ON -DBUILD_SHARED_LIBS=ON` — enables user-written
  Verilog-A device plugins (e.g. RCSJ Josephson).
- `-DCMAKE_CXX_FLAGS='-fopenmp'` — Trilinos/Kokkos linked with OpenMP;
  must propagate to Xyce or link fails.
- `-DADMS_EXECUTABLE=...` — explicit path to admsXml.

Verify after install:
```bash
/data/<user>/xyce_mpi_install/bin/Xyce -v    # expect DEVELOPMENT-YYYYMMDDHHMM-opensource
/data/<user>/xyce_mpi_install/bin/Xyce -capabilities | head -5
# Expect first line: "Parallel with MPI"
```

---

## Stage 5 — Build a Verilog-A plugin (RCSJ example, ~1 min)

```bash
cd /data/<user>
mkdir -p xyce_adms_jj
cd xyce_adms_jj

# Minimal RCSJ Verilog-A — see xyce_adms_jj/rcsj.va for the full model.
# Key parameters: Ic, Rn, Cj, Rph (1TΩ DC-gauge anchor — MANDATORY, see
# feedback_vf_port_termination.md / reference_xyce_landsman5.md).

source /data/<user>/miniforge/etc/profile.d/conda.sh
conda activate xyce-deps
export PATH=/data/<user>/adms_install/bin:/data/<user>/xyce_mpi_install/bin:$PATH
buildxyceplugin.sh -o plugin_build_mpi rcsj.va /data/<user>/xyce_mpi_install

ls plugin_build_mpi/librcsj.so     # expect ~140 kB .so
```

Usage in netlist:
```
Yrcsj J1 p n Ic=3.5e-6 Rn=550 Cj=60e-15
```

And invoke:
```bash
mpirun -np N /data/<user>/xyce_mpi_install/bin/Xyce \
  -plugin /data/<user>/xyce_adms_jj/plugin_build_mpi/librcsj.so \
  netlist.cir
```

---

## Known gotchas (each bit us once)

1. **conda-forge `trilinos` shadows source Trilinos headers.** Its
   `EpetraExt_config.h` lacks `HAVE_BTF`; conda env's `/include/` comes
   BEFORE your `-I/data/.../trilinos_install_mpi/include` in compiler
   search order, so the wrong header wins. Symptom: "HAVE_BTF not declared
   in this scope" during Xyce CMake probe, even though the header at
   `/data/<user>/trilinos_install_mpi/include/EpetraExt_config.h` DOES
   define it. **Fix**: never install conda trilinos. If installed, remove
   it and re-add mpich standalone.

2. **Missing Isorropia causes misleading "BTF missing" errors.** Xyce's
   CMake probes fail in cascade — the ACTUAL missing piece is Isorropia
   (required for MPI Xyce), but CMake reports EpetraExt features as the
   problem. **Fix**: `-DTrilinos_ENABLE_Isorropia=ON` in Trilinos cfg.

3. **conda kokkos-4.5 ships CUDA-enabled by default.** Pulls in
   `CUDAToolkit` dependency that fails on non-GPU builds. **Fix**: if
   using conda kokkos at all (not recommended — build with Trilinos),
   explicitly pin: `kokkos=4.5.01=hbcb5ad4_0` (CPU-only variant).

4. **Parallel-running makes corrupt libxyce.so vtables.** Launching a
   second `make install` in the same build tree while another is running
   produces a non-functional binary with `symbol lookup error: undefined
   symbol: _ZTV...` at runtime. **Fix**: always `pgrep -f make` before
   starting a build; if stale, kill it first.

5. **`nohup bash cfg.sh && make` env-scope bug.** Running `bash cfg.sh`
   in a subshell makes `conda activate` scoped to that subshell only —
   subsequent `make` runs without the env and `mpic++` can't find its
   `x86_64-conda-linux-gnu-c++` backend (error 127). **Fix**: put
   `source conda.sh && conda activate xyce-deps` inside the SAME outer
   `bash -c` block that runs both cfg AND make (see `build_wrapper.sh`
   template above).

6. **bison 3.0.4 (system on RHEL 8) is too old.** Xyce needs bison ≥ 3.3.
   **Fix**: the conda `xyce-deps` env provides 3.8.2 — just make sure
   PATH has conda env first.

7. **Clock skew warnings during install.** landsman filesystems sometimes
   report file mtimes 15 min in the future. `make: warning: Clock skew
   detected` is harmless for this purpose; the build does complete.

8. **HB is HB-only — not `.AC`.** Xyce's `YLIN` Touchstone device is a
   no-op during `.AC` and `.TRAN`; only `.HB` exercises the
   `loadFreqDAEMatrices(frequency, ...)` code path. See
   `reference_xyce_landsman5.md`.

---

## Deploying to a new landsman server

1. SSH test: `ssh landsmanN "hostname; df -h /data"`.
2. Verify `/data/<user>/` has ≥ 50 GB free.
3. Follow Stages 0–5 in order. Each stage has a verification command.
4. Full wall time: ~45 min – 1 h (mostly Trilinos).

**Don't copy a compiled Xyce across servers** unless the new machine has
the same glibc/kernel/conda versions — build from source for reliability.
Binaries reference rpath-linked conda libs.

---

## Fast-path summary (cheat sheet)

Assuming miniforge already present:
```
mamba create -n xyce-deps -c conda-forge suitesparse fftw openblas bison flex cmake gxx_linux-64=13 gfortran_linux-64=13 mpich
# build admsXml from Qucs source       (~2 min)
# download + build Trilinos 16.1 w/ flags above   (~30 min)
# download + build Xyce w/ flags above           (~15 min)
# buildxyceplugin.sh -o plugin_build_mpi rcsj.va /data/<user>/xyce_mpi_install
# mpirun -np N Xyce -plugin ... netlist.cir
```
