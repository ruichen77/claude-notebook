# Palace Install Notes & Cross-Machine Handoff

> **Purpose.** Document how Palace + deps got installed on landsman5 (hard path, RHEL 8 no sudo), and give a fresh Claude Code agent concrete instructions to install it on marika (Pop!_OS, sudo available, RTX 3060 Ti) or any other machine.
>
> **Audience.** Future Claude Code agents. Read the whole thing before starting. The "What broke on landsman5" section is the important part — most of it can be avoided on marika because sudo + Ubuntu are available.

---

## 0. Quick reference — verified install on landsman5

| Item | Value |
|---|---|
| Host | `landsman5` (RHEL 8.10, 256 cores, 1 TiB RAM, NVIDIA A100 80 GB, driver 535.161.08) |
| Palace prefix | `/data/rzhao/spack/opt/spack/linux-zen3/palace-0.16.0-rhkh2gzqr4jbcswcrnhxcj7e3jh5hljs` |
| Env bootstrap | `source /data/rzhao/palace-setup/env.sh && spack load palace` |
| Spack root | `/data/rzhao/spack` |
| Merged gcc@12 tree | `/data/rzhao/gtc12` |
| CUDA toolkit | `/data/rzhao/cuda-12.2` |
| Example configs | `/data/rzhao/palace-examples/examples/` |
| GPU backend verified | `Detected 1 CUDA device` / `libCEED backend: /gpu/cuda/magma` |
| Spec that concretized | `palace@0.16.0 +cuda cuda_arch=80 ~mumps ~strumpack ~arpack ^cuda+allow-unsupported-compilers %gcc@12.2.1` |
| Total packages built | 87 (openmpi, petsc, hypre, mfem, libceed, slepc, magma, superlu-dist, sundials, umpire, camp, blt, ...) |

**Never use `/home` on landsman5** — it's at 100 % utilization from other users. Everything lives under `/data/rzhao`. Saved as memory `feedback_landsman5_workdir.md`.

---

## 1. Intended install pipeline (what it would look like if nothing broke)

```
1. Probe host          → OS, GPU, toolchain, sudo, disk
2. Pick working dir    → enough space, fast I/O if possible
3. Install CUDA        → toolkit (driver already present)
4. Install gcc+gfortran→ from package manager if possible
5. Clone & bootstrap spack
6. Register externals  → gcc, cuda, cmake, mpi
7. spack install palace+cuda cuda_arch=<SM> ~mumps ~strumpack ~arpack
8. Smoke test on GPU   → cpw or spheres example with Device=GPU
9. Document in Notion + memory
```

On marika steps 3 and 4 collapse into `apt install`. On landsman5 they exploded into hours of RPM archaeology.

---

## 2. What broke on landsman5 (so you know what to avoid)

### 2.1. CUDA runfile crashed on `exec -title`
**Symptom.** `./cuda_12.2.2_linux.run --silent --toolkit ...` failed with `exec: -t: invalid option`.
**Root cause.** Makeself 2.1.4 wrapper tries to spawn xterm when `$DISPLAY` is set but `tty -s` is false (headless ssh with DISPLAY forwarded). It runs `exec $XTERM -title "$label" -e "$0"`, and with empty `$XTERM` bash parses `-title` as a flag.
**Fix.** `unset DISPLAY` before invoking the runfile. On marika use the apt/NVIDIA deb package instead → this won't happen.

### 2.2. Transitive Fortran requirement in Spack 1.x
**Symptom.** `spack spec palace` errored with "Only external, or concrete, compilers are allowed for the fortran language" even with `~mumps ~strumpack ~arpack`.
**Root cause.** Spack 1.x treats Fortran as a first-class compiler language; petsc, magma, openmpi+fortran, superlu-dist+fortran, sundials still pull it transitively. Palace's `~mumps ~strumpack ~arpack` only disables Fortran at the Palace level, not deep deps.
**Fix.** Provide a working gfortran. On landsman5 that meant merging an extracted RPM into a fake gcc prefix. On marika: `sudo apt install gfortran`.

### 2.3. Red Hat gcc-toolset-12/13 ships no gfortran by default
**Symptom.** `/opt/rh/gcc-toolset-12/root/usr/bin/gfortran` did not exist.
**Root cause.** gcc-toolset devtools install gcc and g++ but not gfortran (`gcc-toolset-12-gcc-gfortran` is a separate subpackage). And we had no sudo to `dnf install` it.
**Fix.** Downloaded the RPM via `dnf download gcc-toolset-12-gcc-gfortran-12.2.1-7.8.el8_10.x86_64`, extracted with `rpm2cpio | cpio -idm`, created a merged prefix by symlink-tree copying `/opt/rh/gcc-toolset-12/root/usr` into `/data/rzhao/gtc12`, then overlaying the extracted RPM files on top.
**Marika should skip all of this** — `apt install gfortran` gives you a real compiler.

### 2.4. gcc/g++/gfortran must be real files, not symlinks, in the merged prefix
**Symptom.** In the merged tree, gcc resolved its install prefix via `readlink(/proc/self/exe)` and pointed back at `/opt/rh/gcc-toolset-12/...`, so it looked for libgfortran there — and libgfortran wasn't in the system install. Linker errored with `cannot find -lgfortran`.
**Root cause.** gcc uses `/proc/self/exe` → realpath → strip `bin/` to find its install dir. A symlink points gcc at the target's prefix, not the symlink's parent.
**Fix.** `cp` the actual gcc/g++/gfortran ELF files into `/data/rzhao/gtc12/bin/`. Everything else (ld, cc1, as, cpp, lto1, f951, libexec/*) can stay as symlinks — those are looked up relative to gcc's install prefix, which now correctly points into the merged tree.
**Saved as memory feedback** under general "merged gcc prefix" notes if needed.

### 2.5. Red Hat's gcc-toolset-12 libgfortran.a has TPOFF relocations
**Symptom.** Tried to convert the extracted static `libgfortran.a` to a shared `libgfortran.so.5` via `ld --whole-archive`. Failed with `R_X86_64_TPOFF32 against thread_unit: relocation not PIC`.
**Root cause.** Red Hat compiles gcc-toolset libgfortran.a without `-fPIC` because they intend users to dynamically link against the **system** libgfortran.so.5 (which is gcc 8.5.0). You can't turn a non-PIC static library into a shared library.
**Fix.** Grabbed a pre-built `libgfortran.so.5.0.0` from `~/.julia/juliaup/julia-1.12.5+0.x64.linux.gnu/lib/julia/` — Julia 1.12 bundles a modern libgfortran that has `_gfortran_os_error_at` and friends. Copied into `gtc12/lib/gcc/x86_64-redhat-linux/12/libgfortran.so.5.0.0` + symlinks.
**On marika**: `apt install libgfortran5` gives you a real one that matches the system gcc. Non-issue.

### 2.6. System libgfortran.so.5 (gcc 8.5) is missing modern symbols
**Symptom.** After building petsc with gcc 12, slepc failed to link the PETSc test program with `libpetsc.so: undefined reference to _gfortran_os_error_at`.
**Root cause.** `_gfortran_os_error_at` is a runtime call that gcc 10+ emits. RHEL 8's `/usr/lib64/libgfortran.so.5` is from gcc 8.5 and doesn't have it. The SONAME is the same (libgfortran.so.5) but the symbol set differs.
**Fix.** Same as 2.5 — substituted the Julia-bundled modern libgfortran.so.5.
**Marika has recent Ubuntu** → this is automatic.

### 2.7. CUDA 12.2 cudafe++ can't parse gcc 13 libstdc++ headers
**Symptom.** First attempt used gcc 13.2.1. camp@2025.12.0 failed to build with dozens of `error: type name is not allowed` / `identifier "__is_convertible" is undefined` in `/opt/rh/gcc-toolset-13/root/usr/include/c++/13/type_traits`.
**Root cause.** gcc 13's libstdc++ uses newer compiler built-ins (`__is_convertible`, `__remove_reference`, `__reference_converts_from_temporary`) that nvcc 12.2's EDG-based cudafe++ frontend doesn't recognize. `-allow-unsupported-compiler` bypasses the version check but can't fix the parse errors. CUDA 12.4+ would work with gcc 13.
**Fix.** Pivoted to gcc 12.2.1 (officially supported by CUDA 12.2) and still added `+allow-unsupported-compilers` to the CUDA external in packages.yaml (needed for some of Palace's deps that hit their own compiler check).
**On marika**: if CUDA ≥12.4 and gcc ≤13 → fine. If gcc 14+ → use CUDA ≥12.6.

### 2.8. binutils 2.38 ELF alignment bug broke openblas
**Symptom.** openblas built successfully, but `ldd libopenblas.so.0` said `not a dynamic executable`, `LD_PRELOAD=libopenblas.so.0 /bin/true` died with `ELF load command address/offset not properly aligned`. Palace's PETSc test program couldn't run.
**Root cause.** `readelf -l` showed 3 of 4 LOAD segments with `Align = 0x8000` but offsets not properly aligned relative to the p_vaddr modulo p_align. Plain binutils 2.38 bug in gcc-toolset-12. Rebuilding with `ldflags="-Wl,-z,max-page-size=0x1000,-z,common-page-size=0x1000"` did **not** fix it under gcc 12 (probably openblas's Makefile swallows LDFLAGS).
**Fix.** Replaced `/data/rzhao/gtc12/bin/ld`, `/data/rzhao/gtc12/bin/ld.bfd`, and `/data/rzhao/gtc12/libexec/gcc/x86_64-redhat-linux/12/ld` with the binutils 2.40 binary from `/opt/rh/gcc-toolset-13/root/usr/bin/ld`. This is safe because binutils is largely self-contained; 2.40 in gcc-toolset-13 produced correct 0x1000 alignment.
**On marika**: system binutils (likely ≥2.40) is fine. Non-issue.

### 2.9. Spack compiler-wrapper strips `-L /usr/lib64` paths
**Symptom.** Even after symlinking `libgfortran.so` inside `gtc12/lib/gcc/...` to `/usr/lib64/libgfortran.so.5`, the link still failed.
**Root cause.** Spack's `SPACK_SYSTEM_DIRS` filter strips system paths from `-L` flags for reproducibility. Symlinks pointing at `/usr/lib64` get resolved by the linker at a late stage but Spack's wrapper rewrites the library search path.
**Fix.** Copied libgfortran.so.5 as a real file into the merged prefix (avoids any `/usr/lib64` reference).

### 2.10. `NVCC_PREPEND_FLAGS` must be set via Spack's `config:build_env`
**Symptom.** `+allow-unsupported-compilers` variant on the CUDA external didn't propagate to CMake's `enable_language(CUDA)` probe — camp's BLT CMake failed "gcc versions later than 12 are not supported".
**Root cause.** The Spack variant only adds the flag to nvcc's own invocations, not to CMake's CUDA compiler identification probe.
**Fix.** `spack config --scope=user add config:build_env:NVCC_PREPEND_FLAGS:-allow-unsupported-compiler`. This makes the env var set for every package's build, so nvcc picks it up in CMake's probe.
**Keep this on any gcc>12 + CUDA<12.4 install.**

---

## 3. landsman5 install — final state of config files

### 3.1. `/data/rzhao/palace-setup/env.sh`
```bash
# Source this to activate Palace/Spack environment on landsman5
source /opt/rh/gcc-toolset-13/enable     # for build tools; gcc compiler is overridden below via SPACK
export SPACK_USER_CONFIG_PATH=/data/rzhao/spack-user/config
export SPACK_USER_CACHE_PATH=/data/rzhao/spack-user/cache
export CUDA_HOME=/data/rzhao/cuda-12.2
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
source /data/rzhao/spack/share/spack/setup-env.sh
export NVCC_PREPEND_FLAGS="-allow-unsupported-compiler"
```

### 3.2. `/data/rzhao/spack-user/config/packages.yaml`
```yaml
packages:
  all:
    providers:
      mpi: [openmpi]
  gcc:
    externals:
    - spec: gcc@12.2.1 languages:='c,c++,fortran'
      prefix: /data/rzhao/gtc12
      extra_attributes:
        compilers:
          c: /data/rzhao/gtc12/bin/gcc
          cxx: /data/rzhao/gtc12/bin/g++
          fortran: /data/rzhao/gtc12/bin/gfortran
    - spec: gcc@13.2.1 languages:='c,c++,fortran'
      prefix: /data/rzhao/gtc13
      extra_attributes:
        compilers:
          c: /data/rzhao/gtc13/bin/gcc
          cxx: /data/rzhao/gtc13/bin/g++
          fortran: /data/rzhao/gtc13/bin/gfortran
    - spec: gcc@8.5.0 languages:='c,c++'
      prefix: /usr
      extra_attributes:
        compilers:
          c: /usr/bin/gcc
          cxx: /usr/bin/g++
  cuda:
    externals:
    - spec: cuda@12.2.140+allow-unsupported-compilers
      prefix: /data/rzhao/cuda-12.2
    buildable: false
  cmake:
    externals:
    - spec: cmake@3.26.5
      prefix: /usr
    buildable: false
  openmpi:
    variants: +cuda fabrics=auto
  palace:
    require:
    - spec: '%gcc@12.2.1'
```

### 3.3. `spack install` invocation (the one that worked)
```bash
source /data/rzhao/palace-setup/env.sh
spack install --fail-fast -j 32 \
    palace@0.16.0 +cuda cuda_arch=80 ~mumps ~strumpack ~arpack \
    ^cuda+allow-unsupported-compilers
```

---

## 4. Marika install plan (for a fresh agent)

### 4.1. Host profile (known from CLAUDE.md, needs re-verification at start)

| Item | Expected value | Command to verify |
|---|---|---|
| OS | Pop!_OS (Ubuntu-based, likely 22.04 or 24.04) | `cat /etc/os-release` |
| CPU | 32 vCPU | `nproc` |
| RAM | ~124 GiB | `free -h` |
| GPU | NVIDIA RTX 3060 Ti (8 GB, Ampere, **sm_86**) | `nvidia-smi -L` |
| Driver | ? | `nvidia-smi --query-gpu=driver_version --format=csv,noheader` |
| sudo | Yes (home server) | `sudo -n true && echo ok` |
| Disk | Pick the largest local SSD partition | `df -h ~ /opt /data /mnt` |

### 4.2. Working directory convention
Before starting, **ASK THE USER** where to put the install. Default suggestion: `~/palace-setup` + `~/spack` (both under `$HOME`). If the user has a dedicated SSD mount (e.g. `/mnt/nvme/rzhao` or similar), use that. **Do NOT assume `/data/rzhao` exists** — that's landsman5-specific.

### 4.3. Step-by-step

#### Step 1 — probe the host
```bash
ssh marika 'uname -a; cat /etc/os-release | head -4; nproc; free -h | head -2; nvidia-smi -L; nvidia-smi --query-gpu=driver_version,memory.total --format=csv; which nvcc gcc gfortran cmake mpicc git python3 gmsh spack; gcc --version | head -1; cmake --version | head -1; df -h ~ /tmp; sudo -n true && echo sudo_ok || echo sudo_needs_password'
```
**Note.** From a Claude Code sandboxed bash, private-network IPs may be refused. If so, ask the user to run the command with `!` prefix so output lands in the session.

#### Step 2 — install system toolchain via apt
```bash
sudo apt update
sudo apt install -y \
    build-essential gfortran \
    cmake git python3 python3-pip python3-venv \
    libopenmpi-dev openmpi-bin \
    libopenblas-dev libopenblas-openmp-dev liblapack-dev \
    libblas-dev pkg-config \
    gmsh libgmsh-dev
```

This single command collapses **all of landsman5 sections 2.2–2.8**. Confirm gfortran works:
```bash
gfortran --version   # should print something
echo 'program hi; print *, 42; end program' > /tmp/hi.f90
gfortran /tmp/hi.f90 -o /tmp/hi && /tmp/hi && rm /tmp/hi*
```

#### Step 3 — CUDA toolkit
Check if already installed: `nvcc --version`. If missing or too old for a Palace+cuda build:

```bash
# Ubuntu 22.04 or 24.04, use NVIDIA's apt repo
distro=$(lsb_release -rs | sed 's/\.//')    # e.g. 2204
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu${distro}/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-4    # match the driver; bump if needed
```

**Target CUDA version vs driver:**
- RTX 3060 Ti with driver ≥525.xx → CUDA 12.0 OK
- Driver ≥535.xx → CUDA 12.2
- Driver ≥550.xx → CUDA 12.4
- Driver ≥560.xx → CUDA 12.6
- **Prefer CUDA 12.4+** to avoid the landsman5 gcc-13 incompatibility (§2.7). If the driver is too old, update it first (`sudo apt install nvidia-driver-550` or similar) and reboot.

After install, add `/usr/local/cuda-XX.Y/bin` to PATH and `lib64` to LD_LIBRARY_PATH. Verify with `nvcc --version`.

#### Step 4 — clone and bootstrap Spack
```bash
git clone --depth=1 -c feature.manyFiles=true https://github.com/spack/spack.git ~/spack
echo 'source ~/spack/share/spack/setup-env.sh' >> ~/.bashrc     # optional
source ~/spack/share/spack/setup-env.sh
spack compiler find            # picks up system gcc
spack external find cmake cuda openmpi openblas pkgconf   # picks up apt-installed deps
```

`spack external find` auto-detects installed packages and writes a `packages.yaml`. Much cleaner than manually writing YAML like we did on landsman5.

#### Step 5 — dry-run concretize to spot issues early
```bash
spack spec -I palace@0.16.0 +cuda cuda_arch=86 ~mumps ~strumpack ~arpack
```
Look for:
- All deps should say `[+]` (already installed from apt externals) or will build
- Compiler should resolve to system `gcc@<something>`
- No Fortran errors
- No "Only external, or concrete, compilers" messages
- CUDA conflict? → add `^cuda+allow-unsupported-compilers` if the gcc/CUDA combo is unsupported

**The spec differs from landsman5 in `cuda_arch=86`** (RTX 3060 Ti is Ampere SM 8.6, not 8.0 like A100).

#### Step 6 — install
```bash
# ~124 GB RAM and 32 cores → -j 16 is plenty, leave headroom for the compiler
spack install --fail-fast -j 16 \
    palace@0.16.0 +cuda cuda_arch=86 ~mumps ~strumpack ~arpack
```
Expected time: 1–2 hours (fewer cores than landsman5 but simpler dep chain since no compiler bootstrap).

**Run as a background job with nohup, save to a logfile, poll from a watcher.** Same pattern as landsman5:
```bash
nohup spack install ... > ~/palace-setup/spack-install.log 2>&1 &
disown
```

#### Step 7 — Python mesh toolchain
```bash
python3 -m venv ~/palace-setup/venv
source ~/palace-setup/venv/bin/activate
pip install pygmsh meshio gmsh
```
(The `gmsh` pip package is the same Gmsh SDK but pip-shipped; `libgmsh-dev` from apt is fine too and lets you use `import gmsh`.)

#### Step 8 — examples + smoke test
```bash
cd ~ && git clone --depth 1 --branch v0.16.0 --filter=blob:none --sparse \
    https://github.com/awslabs/palace.git palace-examples
cd palace-examples && git sparse-checkout set examples

source ~/spack/share/spack/setup-env.sh && spack load palace
cd examples/spheres
sed -i 's/"Device": "CPU"/"Device": "GPU"/' spheres.json
time palace -np 1 spheres.json 2>&1 | tee /tmp/palace-smoke.log
grep -iE "detected.*cuda|device configuration|libceed backend" /tmp/palace-smoke.log
```

Expected output on success:
```
Detected 1 CUDA device
Device configuration: cuda,cpu
Memory configuration: host-umpire,cuda-umpire
libCEED backend: /gpu/cuda/magma
```

If it runs to completion and prints an energy value, call it done.

#### Step 9 — write env.sh and README
Put a `~/palace-setup/env.sh` analogous to landsman5's, updating paths for marika. Put a short `~/palace-setup/README.md` listing install prefix, env bootstrap command, example location.

#### Step 10 — update Notion + memory
- Add a note to the Palace Notion tutorial page (under "Where it's installed") with marika paths.
- Add a memory entry `feedback_marika_workdir.md` listing the canonical working directory on marika.

### 4.4. Gotchas specific to marika vs landsman5

| Risk | Why | Mitigation |
|---|---|---|
| **8 GB VRAM on 3060 Ti** | Palace driven-mode >1 M DOFs easily exceeds 8 GB → OOM kill | Fall back to CPU for big sims; GPU is fine for eigenmode & medium driven work. Document in the README. |
| Ubuntu **gcc too new** | Pop!_OS ships gcc 13 or 14; if CUDA toolkit is <12.4 → §2.7 replay | Install CUDA 12.4+ OR install an older gcc (`apt install gcc-12 g++-12 gfortran-12`) and tell Spack to use it |
| Driver/toolkit mismatch | Pop!_OS may auto-upgrade driver to something the installed CUDA can't use | Check with `nvidia-smi` + `nvcc --version`; NVIDIA doc has the matrix |
| **Disk space** | 124 GB RAM is generous, but `~` or `/home` may be on a small partition | Pick the largest SSD; Spack install tree ≈15 GB + CUDA toolkit ≈7 GB + staging ≈10 GB = **~35 GB minimum** |
| openmpi from apt is **not CUDA-aware** | apt openmpi doesn't build with `--with-cuda` | Either use it anyway (fine for CPU ranks) OR let Spack rebuild openmpi with `+cuda` — adds ~15 min |
| Sim outputs on a network drive | If marika has NAS mount, writing postpro there slows IO a lot | Keep outputs on local SSD |

### 4.5. Useful commands while debugging the install

```bash
# see what's currently running in the spack install
ps -o pid,pcpu,etime,cmd -C python3 | grep "spack install"

# tail the log
tail -f ~/palace-setup/spack-install.log

# count packages built so far
source ~/spack/share/spack/setup-env.sh && spack find | grep -c "^[a-z0-9]"

# find the most recent failed package
grep -E "^\[x\]" ~/palace-setup/spack-install.log | tail -5

# dump the build env for a failed package (to check -L and -l flags)
cat /tmp/$USER/spack-stage/spack-stage-<pkg>-*/spack-build-env.txt | grep -iE "LIBRARY|LDFLAGS|SPACK_"
```

---

## 5. What to ask the user before starting

1. **Where to install** — `~/palace-setup`, `~/spack`? Or a dedicated partition?
2. **CUDA version preference** — latest toolkit vs match existing (if any)?
3. **GPU or CPU build** — GPU is the point, but confirm before committing to the extra 30 min for openmpi+cuda and CUDA toolkit install.
4. **Rebuild openmpi with +cuda** — yes (GPU-aware MPI, more setup) or reuse apt openmpi (simpler, CPU-only MPI)?
5. **Is Palace on marika intended for TWPA/qubit sims or just practice?** — affects whether we enable AMR, high-order elements, etc. in the defaults.

Default answers if the user says "just do it":
- `~/palace-setup` and `~/spack`
- Latest CUDA compatible with driver
- GPU build
- Reuse apt openmpi (faster setup, swap later if needed)

---

## 6. How to test if install succeeded

Required successes (in order):
1. `nvcc --version` prints a CUDA version ≥12.0
2. `gfortran --version` prints gcc ≥11
3. `spack spec palace+cuda` concretizes without error
4. `spack install palace+cuda` exits 0
5. `spack load palace && palace --help` works
6. `palace --dry-run spheres.json` exits 0
7. `palace -np 1 spheres_gpu.json` runs and prints `Detected 1 CUDA device`
8. `libCEED backend: /gpu/cuda/magma` appears in output
9. Final memory report < 8 GB (fits in 3060 Ti)

If 7 fails with CUDA init errors → check `nvidia-smi` + permissions. If 8 shows `/cpu/self/...` → GPU device not enabled in config JSON; check `Device` field.

---

## 7. Final deliverables checklist

- [ ] Palace binary at `<prefix>/bin/palace` runs `--help`
- [ ] env.sh bootstrap script that sources spack + sets CUDA paths
- [ ] README.md with install location, env bootstrap, example dir, GPU-verified line
- [ ] Example configs cloned to `~/palace-examples/`
- [ ] Smoke test passed (spheres or cpw with Device=GPU)
- [ ] Notion tutorial page updated with marika paths
- [ ] New memory `feedback_marika_workdir.md` with working-dir convention
- [ ] Commit the handoff doc (this file) updates if you discovered new gotchas

---

## 8. References

- Landsman5 install session log: search for `ssh landsman5` in this project's Claude Code transcripts (dates 2026-04-10 and 2026-04-11)
- Notion tutorial page: search "Palace — AWS Full-Wave EM Simulator: Usage Tutorial & Reference" (page id `33f6a718-2abd-816a-95f8-f9e08b595268`)
- Palace docs: https://awslabs.github.io/palace/stable/
- Spack docs: https://spack.readthedocs.io/en/latest/
- Gmsh docs: https://gmsh.info/doc/texinfo/gmsh.html
- CUDA compatibility matrix: https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html
- Palace paper: Reid et al., SoftwareX 2024, https://doi.org/10.1016/j.softx.2024.101634
