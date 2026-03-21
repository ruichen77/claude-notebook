# Wave-Domain Co-Simulation Framework

## Overview

The cosim package couples **JoSIM transient TWPA simulations** with **S-parameter filter blocks** to model a cascaded BPF-TWPA-BPF system in the time domain. Instead of embedding filter netlists directly into JoSIM (which would be extremely slow for realistic filter models), the framework decomposes the problem into traveling-wave signals at each TWPA boundary and iterates between JoSIM (nonlinear TWPA) and convolution-based filter scattering (linear S-parameter blocks).

**Use cases:**
- Studying how bandpass filters affect TWPA gain shape and spur suppression
- Modeling real diplexer (DPX) effects on gain bandwidth
- Validating pump-off passive cascades against analytic scikit-rf predictions
- Exploring filter design parameters (passband, rejection) on system performance

### System Topology Options

1. **BPF-TWPA-BPF** (full sandwich): Both input and output have BPFs. Used for pump-off validation.
2. **TWPA-BPF** (output filter only): Input BPF replaced with perfect thru (S21=1). Pump and signal enter TWPA directly; only output is filtered. Most common topology for gain studies.
3. **DPX-TWPA-BPF**: Input uses a real diplexer S-parameter file. Pump enters via one DPX port, signal via another. (Planned; requires 3-port or pre-computed pump injection.)

## Architecture

### Module Breakdown

```
cosim/
  __init__.py               # Package init (v0.1.0)
  sparam_kernel.py          # Module 1: S2P -> time-domain impulse response kernels
  wave_io.py                # Module 2: Wave extraction/injection at TWPA boundaries
  convolution_engine.py     # Module 3: Filter scattering via time-domain convolution
  cosim_orchestrator.py     # Module 4: Main iteration loop (the heart of co-sim)
  vna_extract.py            # Module 5: S21 extraction from converged waveforms
  sweep_runner.py           # Module 6: Pump power sweep automation
  synthetic_bpf.py          # Synthetic absorptive BPF generator (Butterworth)
  config.yaml               # Example configuration file
  run_cosim_pumpoff.py      # Local pump-off validation driver
  utils/
    josim_runner.py          # JoSIM subprocess wrapper
    plotting.py              # Convergence, S21, wave, and kernel plots
    pwl_writer.py            # PWL file read/write/inline-format utilities
  tests/
    test_convolution.py
    test_kernels.py
    test_passive.py
    test_wave_io.py
```

### Data Flow Diagram

```
                        ITERATION LOOP
                        ==============

  External Source           BPF_in (kernels_in)         TWPA (JoSIM)           BPF_out (kernels_out)        Load
  ================         ===================         ============           ====================         ====
  pump + signal     a1-->  [S11 S12]  b2(=b_left)-->   [JoSIM .cir]  a_right-->  [S11 S12]  b2(=b_load)-->  matched
  (voltage wfm)     <--b1  [S21 S22]  <--a2(=a_left)   [transient]   <--b1(=b_right) [S21 S22]  <--a2       Z0
                     |                     |              |     |                       |
                     |                     +----PWL_left--+     +----PWL_right-----------+
                     |                          (inject)              (inject)
                     v
              b_ext_reflect
              (reflected back
               to source)

  Each iteration:
    1. JoSIM runs with current PWL injection files
    2. Extract a_left, a_right from JoSIM CSV (port voltages & currents)
    3. Resample from dt_sim (0.1 ps) to dt_kernel (1 ps)
    4. Scatter through BPF_in:  b_left_new  = S12*a1_source + S22*a_left
    5. Scatter through BPF_out: b_right_new = S11*a_right   + S12*a2_load
    6. Under-relax: b = alpha*b_new + (1-alpha)*b_prev
    7. Write updated PWL files, re-patch netlist
    8. Check convergence: eps = max(eps_left, eps_right)
    9. If eps < threshold: done. Else: goto 1.
```

### Port Naming Convention

| Port name | Location | TWPA node | BPF port |
|-----------|----------|-----------|----------|
| `left`    | TWPA input  | `n1` | BPF_in port 2 |
| `right`   | TWPA output | `n{ncells+1}` (e.g., `n201`) | BPF_out port 1 |

## Key Concepts

### Wave Decomposition at TWPA Boundaries

At each TWPA-filter boundary, a **port circuit** is inserted into the JoSIM netlist:

```
V_inj_<port> -- R_<port>(Z0) -- TWPA_node -- (TWPA interior)
```

The port circuit has a voltage source `V_inj` in series with a resistor `R = Z0`. This lets us both inject waves into the TWPA and extract waves coming out of it.

**Wave definitions (co-sim convention):**

```
a(t) = V_inj / (2*sqrt(Z0)) - sqrt(Z0) * I_R    (wave FROM TWPA toward filter)
b(t) = V_inj / (2*sqrt(Z0)) + sqrt(Z0) * I_R    (wave FROM filter into TWPA)
```

where `I_R` is the current through `R_<port>`, positive from `V_inj` toward TWPA node.

**IMPORTANT sign convention note:** The `a` and `b` labels are swapped from the standard power-wave convention. In this codebase, `a_t` means the wave traveling FROM the TWPA INTO the filter (outgoing from TWPA), and `b_t` means the wave traveling FROM the filter INTO the TWPA (incoming to TWPA). This is documented in `wave_io.py` lines 293-303.

**Injection voltage:** `V_inj(t) = 2 * sqrt(Z0) * b(t)` -- the backward-traveling wave `b(t)` is what gets injected into the TWPA via the PWL source.

### S-Parameter Kernels

S-parameter files (.s2p) are converted to time-domain impulse response kernels for convolution:

1. **Load S2P** via scikit-rf
2. **Enforce passivity**: SVD-based clipping of singular values > 1 at each frequency
3. **Extrapolate and pad**: Cosine taper beyond measured band, zero-pad to f_Nyquist = 1/(2*dt)
4. **IFFT**: `np.fft.irfft` for real-valued impulse response
5. **Causality check**: Verify acausal energy (second half of array) is negligible. If >10%, roll the kernel to enforce causality.
6. **Zero second half**: Set h[n_time//2:] = 0 to eliminate wrap-around artifacts
7. **Truncate** at -60 dB of peak (with 10% margin)

All four kernels (s11, s12, s21, s22) are truncated to a common length.

### Under-Relaxation

To prevent oscillations in the iterative loop, the scattered waves are under-relaxed:

```
b_new = alpha * b_computed + (1 - alpha) * b_prev
```

- `alpha = 1.0`: No relaxation (fastest convergence if stable)
- `alpha = 0.5`: Default -- average of new and previous (good stability)
- `alpha = 0.3`: More conservative (use if diverging)

### Convergence Criterion

```
eps = max(eps_left, eps_right)
eps_port = ||b_new - b_prev||_2 / ||b_new||_2
```

Convergence is declared when `eps < convergence_threshold` (default 1e-3). Typical convergence: 3-5 iterations for pump-off, 5-10 for pump-on.

**Divergence detection**: If epsilon increases for `divergence_window` (default 3) consecutive iterations, a warning is logged. Consider reducing `relaxation_alpha`.

## How to Run

### Prerequisites

- **JoSIM binary**: `/data/rzhao/jtwpa_campaign/tools_josim/bin/josim-cli`
- **Python**: anaconda3 on landsman2/3/4 (`python3`), or `/home/US8J4928/base/bin/python3` on landsman5
- **Python packages**: numpy, scipy, scikit-rf, pyyaml, matplotlib (all available in anaconda)
- **Cosim package**: `/data/rzhao/repos/cosim/` (on shared NFS, accessible from all landsman servers)
- **TWPA netlist generator**: `/data/rzhao/jtwpa_campaign/generate_jtwpa_chirped.py`

### Quick Start: Pump-Off Validation

This runs the BPF-TWPA-BPF cascade with no pump to verify the framework produces correct passive S21.

```bash
# On a landsman server (e.g., landsman2)
cd /data/rzhao/repos/cosim
python3 -u run_cosim_pumpoff.py > /data/rzhao/results/cosim_pumpoff.log 2>&1
```

**What it does** (step by step):
1. Generates two synthetic BPF S2P files (6-8 GHz passband, order-5 Butterworth, 0.5 dB IL)
2. Generates a TWPA netlist with zero pump (`--ppump -999`)
3. Creates a multi-tone CW source (5 tones: 6.0, 6.5, 7.0, 7.5, 8.0 GHz at 1 uV each)
4. Computes analytic cascade S21 via scikit-rf for comparison
5. Runs iterative co-simulation (max 10 iterations, alpha=0.5, threshold=1e-3)
6. Extracts S21 at tone frequencies from converged waves

**Results**: Saved to `/data/rzhao/results/YYYYMMDD_HHMM_landsman2_cosim_pumpoff/`

### Running a Pump Sweep (TWPA + Output BPF)

The `run_twpa_bpf_sweep.py` driver runs co-simulation at multiple pump powers with only an output BPF (input is a perfect thru):

```bash
cd /data/rzhao/repos/cosim
python3 -u run_twpa_bpf_sweep.py > /data/rzhao/results/twpa_bpf_sweep.log 2>&1
```

**Key configuration** (hardcoded at top of script):
- `NCELLS = 200` (fast test; use 1956 for realistic)
- `F_PUMP = 8.4e9` Hz
- `PPUMP_VALUES = [-90, -68, -66, -64, -62, -60, -58]` dBm
- 13 signal tones from 4.0 to 10.0 GHz (0.5 GHz spacing)
- Input: perfect thru S2P (S21=1 everywhere)
- Output: synthetic absorptive BPF (6-8 GHz passband)

**Output structure**:
```
results/YYYYMMDD_HHMM_<server>_twpa_bpf_sweep/
  thru.s2p              # Perfect thru input
  bpf_out.s2p           # Synthetic BPF output
  twpa_netlist/         # Generated TWPA .cir
  ppump_-90dBm/         # Per-power-point results
    cosim_work/         # Iteration data
      iter_001/         # JoSIM CSV + PWL for each iteration
      iter_002/
      ...
      final_waves.npz   # Converged wave arrays
      cosim_summary.json
    result.json         # S21 at tones for this power
  ppump_-68dBm/
  ...
  sweep_summary.json    # All results combined
  s21_vs_freq.png       # S21 vs frequency plot
  gain_vs_freq.png      # Gain relative to -90 dBm baseline
  peak_gain_vs_ppump.png
  run.log               # Full log
```

### Using Real S-Parameter Files (DPX-derived BPF)

The `run_twpa_realdpx_sweep.py` driver uses a real diplexer S-parameter file (`Harbord_DPX_as_BPF.s2p`) instead of a synthetic BPF:

```bash
cd /data/rzhao/repos/cosim
python3 -u run_twpa_realdpx_sweep.py > /data/rzhao/results/twpa_realdpx.log 2>&1
```

**Critical step**: Real S-parameter files often have non-uniform (log-spaced) frequency grids. The `resample_bpf_s2p()` function automatically resamples to a uniform 10 MHz grid from DC to 20 GHz. Without this, `sparam_kernel.py` computes a tiny `df` from the minimum frequency spacing and allocates an enormous kernel that exhausts memory.

**Real BPF S2P location**: `/data/rzhao/repos/cosim/Harbord_DPX_as_BPF.s2p`

### Deriving a 2-Port BPF from a 4-Port DPX

To use a 4-port diplexer S4P as a 2-port BPF, terminate the unused ports with matched loads (50 Ohm) and extract the relevant 2-port submatrix. Use scikit-rf:

```python
import skrf
dpx = skrf.Network("diplexer.s4p")
# Extract ports 1,4 (signal path), terminate ports 2,3
bpf = dpx.subnetwork([0, 3])  # 0-indexed: ports 1 and 4
bpf.write_touchstone("dpx_as_bpf.s2p")
```

Then resample to a uniform grid before passing to the cosim framework.

### Running via tmux (Recommended)

For long-running sweeps, use tmux:

```bash
ssh -T landsman2 "tmux new -d -s cosim_sweep 'cd /data/rzhao/repos/cosim && python3 -u run_twpa_bpf_sweep.py > /data/rzhao/results/cosim_sweep.log 2>&1'"
```

**Monitor:**
```bash
ssh -T landsman2 "tail -20 /data/rzhao/results/cosim_sweep.log"
```

## Configuration Reference

All parameters can be set in a YAML config file or passed directly as a dict to `CoSimOrchestrator`. See `/Users/ruichenzhao/projects/twpa/cosim/config.yaml` for the annotated example.

### Core Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `work_dir` | str | `"."` | Working directory for all iteration data |
| `netlist_path` | str | required | Path to base TWPA netlist (.cir) |
| `bpf_in_s2p` | str | required | S2P file for input BPF |
| `bpf_out_s2p` | str | required | S2P file for output BPF |
| `josim_bin` | str | see below | Path to josim-cli binary |
| `josim_timeout` | float | 600.0 | JoSIM timeout in seconds |

### Circuit Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `z0` | float | 50.0 | Port impedance (Ohm) |
| `ncells` | int | 200 | Number of JJ cells in TWPA |
| `input_node` | str | `"n1"` | TWPA input node name |
| `output_node` | str | `"n{ncells+1}"` | TWPA output node name |

### Timing Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `t_stop` | float | 200e-9 | Total simulation time (s) |
| `dt_sim` | float | 0.1e-12 | JoSIM internal timestep (s) |
| `dt_kernel` | float | 1e-12 | Kernel/wave timestep (s). Determines PWL file resolution and convolution grid. |

### Iteration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_iterations` | int | 20 | Maximum co-sim iterations |
| `convergence_threshold` | float | 1e-3 | Relative L2 norm change threshold |
| `relaxation_alpha` | float | 0.5 | Under-relaxation factor (0, 1]. Lower = more stable. |
| `divergence_window` | int | 3 | Consecutive eps increases before warning |

### VNA Extraction Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `t_skip_extract` | float | 50e-9 | Skip initial transient for FFT analysis |
| `window_type` | str | `"tukey"` | Window function for FFT |
| `window_alpha` | float | 0.05 | Tukey window shape parameter |

### Sweep Parameters (sweep_runner.py)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fpump` | float | 8.4e9 | Pump frequency (Hz) |
| `ppump_range_dBm` | list | required | Pump powers for sweep |
| `signal_freqs_GHz` | list | optional | Signal tone frequencies |
| `signal_level_dB` | float | -40 | Signal level relative to pump (dB) |
| `n_sweep_workers` | int | 1 | Parallel workers (1 = sequential) |

## Known Issues and Fixes Applied

### 1. Inline PWL (JoSIM does not support `PWL(file:path)`)

JoSIM cannot load PWL data from external files. The `patch_netlist()` function reads the PWL file content and embeds it inline in the netlist:
```
V_inj_left np_left 0 PWL(t1 v1 t2 v2 ...
+ t3 v3 t4 v4 ...
+ ...)
```
Continuation lines start with `+`. This can make netlists very large (>10 MB for 200k points at dt_kernel=1ps over 200ns). The function decimates to `max_points=25000` if needed.

### 2. Wave Extraction Sign Convention

The `a_t` and `b_t` waves use a **swapped convention** from standard power waves. See `wave_io.py` lines 293-303:
- `a_t` (co-sim "forward") = wave FROM TWPA toward filter = standard `b` (reflected from TWPA)
- `b_t` (co-sim "backward") = wave FROM filter into TWPA = standard `a` (incident into TWPA)

This was a source of bugs early on. The current code is correct and self-consistent.

### 3. BPF_in Port Mapping

In the orchestrator, the BPF_in scattering call maps:
- `a1 = source_a1_in` (external source -> BPF_in port 1)
- `a2 = a_left_k` (TWPA left output -> BPF_in port 2)
- `b2 = b_left_new` (BPF_in port 2 -> TWPA, the wave we inject)

An earlier bug had `b1` and `b2` swapped, causing the reflected wave to be injected instead of the transmitted wave. This is fixed.

### 4. Non-Uniform S-Parameter Frequencies

Real S-parameter files (e.g., from VNA or diplexer datasheets) often have non-uniform frequency grids (log-spaced or adaptive). The `sparam_kernel.py` module uses `df = f[1] - f[0]` to set the output grid. If the first two points are very close (e.g., 100 Hz apart), this creates an enormous kernel that exhausts memory.

**Fix**: Always resample real S2P files to a uniform grid (10 MHz recommended) before passing to the cosim. See `resample_bpf_s2p()` in `run_twpa_realdpx_sweep.py` for the pattern.

### 5. Synthetic BPF Causality

Early versions used a Hilbert transform to compute S21 phase for the synthetic BPF, which produced non-causal kernels when `|S21|` passed through zero (log(0) singularity in the Hilbert integral).

**Fix**: Use `scipy.signal.butter(analog=True)` + `scipy.signal.freqs()` to evaluate the analog transfer function H(jw) directly. This gives a naturally causal, minimum-phase response with correct group delay. The Hilbert transform is only used for S11 phase, where `|S11|` is always ~0.18 (absorptive filter) and never zero.

### 6. JoSIM `-m` Flag Bug

On some servers, large netlists (~26K lines with Debye loss elements) cause JoSIM to silently exit with return code 0 but produce no output CSV when the `-m` (minimal output) flag is used. If this happens, try without `-m` or use a different server.

## Performance

| Configuration | Time per Iteration | Typical Iterations | Total Time |
|---------------|-------------------|-------------------|------------|
| 200 cells, 200 ns | ~3 min | 3-5 | ~10-15 min |
| 1956 cells, 200 ns | ~30 min | 3-5 | ~2-3 hours |
| 1956 cells, 800 ns | ~2 hours | 3-5 | ~6-10 hours |

**Bottleneck**: JoSIM transient simulation dominates runtime. Convolution is fast (seconds).

**Parallelization strategy**: Each pump power point is independent. Split across multiple landsman servers:
- Each server runs its own driver script with a subset of pump powers
- All share `/data/rzhao/` NFS, so results are accessible from anywhere
- Use tmux sessions on each server

**Memory**: Each JoSIM run for 200 cells uses ~200 MB RAM. 1956 cells uses ~2 GB. The cosim orchestrator keeps wave arrays in memory (~50 MB for 200k-point waveforms).

## Best-Match TWPA Parameters

For realistic TWPA simulations that match HDRP094A experimental data, use the parameters documented in the best-match reference file. Key highlights:

- **ncells**: 1956 (not 200 -- 200 is only for quick tests)
- **fpump**: 8.13 GHz
- **Dielectric loss**: Injected via Debye model post-hoc (not via `--tan-delta`)
- **RCSJ**: R0=5000, Rn=549.8 (Ambegaokar-Baratoff derived)
- **RPM stub Cr**: 281 fF (not default 260 fF)
- **dt**: 0.25 ps for 1956-cell runs
- **tstop**: 800 ns for realistic settling

See `memory/reference_josim_best_match_params.md` for the complete 4-step reproduction recipe including generator patching, Debye loss injection, and postprocessing.

**Note**: The current cosim driver scripts use 200 cells for development speed. To switch to realistic 1956-cell runs, change `NCELLS` and adjust `T_STOP`, `DT_SIM`, and the generator flags accordingly. Also apply the Debye loss injection step to the generated netlist before running the cosim.

## API Quick Reference

### Running a cosim programmatically

```python
import sys
sys.path.insert(0, "/data/rzhao/repos")

from cosim.cosim_orchestrator import CoSimOrchestrator
import numpy as np

config = {
    "work_dir": "/data/rzhao/results/my_cosim/",
    "netlist_path": "/path/to/twpa.cir",
    "bpf_in_s2p": "/path/to/bpf_in.s2p",
    "bpf_out_s2p": "/path/to/bpf_out.s2p",
    "josim_bin": "/data/rzhao/jtwpa_campaign/tools_josim/bin/josim-cli",
    "z0": 50.0,
    "ncells": 200,
    "dt_sim": 0.1e-12,
    "t_stop": 200e-9,
    "dt_kernel": 1e-12,
    "max_iterations": 15,
    "convergence_threshold": 1e-3,
    "relaxation_alpha": 0.5,
}

orch = CoSimOrchestrator(config)

# Set source waveform (pump + signal)
n_pts = int(200e-9 / 1e-12) + 1
t = np.linspace(0, 200e-9, n_pts)
pump_V = Vpeak * np.sin(2 * np.pi * fpump * t)
signal_V = Vsig * np.sin(2 * np.pi * fsig * t)
orch.set_source_waveform(time_s=t, pump_V=pump_V, signal_V=signal_V)

# Run
summary = orch.run()
# summary keys: converged, n_iterations, epsilon_history, work_dir
```

### Generating a synthetic BPF

```python
from cosim.synthetic_bpf import generate_synthetic_bpf

generate_synthetic_bpf(
    output_path="my_bpf.s2p",
    f_low=6.0e9,       # passband lower edge
    f_high=8.0e9,      # passband upper edge
    order=5,            # Butterworth order
    insertion_loss_dB=0.5,
    s11_max_dB=-15.0,   # absorptive: S11 < -15 dB everywhere
    n_points=2001,
)
```

### Extracting S21 from results

```python
from cosim.vna_extract import extract_from_cosim

freqs, s21_dB, s21_phase = extract_from_cosim(
    "final_waves.npz",
    z0=50.0,
    t_skip=50e-9,
    freq_range=(4e9, 10e9),
)
```

## File Paths Summary

| Item | Path |
|------|------|
| Cosim package (local) | `/Users/ruichenzhao/projects/twpa/cosim/` |
| Cosim package (server) | `/data/rzhao/repos/cosim/` |
| JoSIM binary | `/data/rzhao/jtwpa_campaign/tools_josim/bin/josim-cli` |
| TWPA netlist generator | `/data/rzhao/jtwpa_campaign/generate_jtwpa_chirped.py` |
| Debye loss injector | `/data/rzhao/jtwpa_campaign/results_campaign5/debye_loss_test/inject_debye_loss.py` |
| Real DPX S2P | `/data/rzhao/repos/cosim/Harbord_DPX_as_BPF.s2p` |
| Best-match params | `~/.claude/projects/-Users-ruichenzhao-projects-twpa/memory/reference_josim_best_match_params.md` |
| Example config | `/Users/ruichenzhao/projects/twpa/cosim/config.yaml` |
