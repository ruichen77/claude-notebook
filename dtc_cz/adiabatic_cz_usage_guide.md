# adiabatic_cz Package — Usage Guide

**Repository**: `git@github.ibm.com:Ruichen-Zhao/dtc_adiabatic_sim.git`
**Primary server**: landsman2 (28 cores, 1 TB RAM)
**Dimension**: 432 (Q0×DTC×Q1 = 6×12×6, hierarchical)

---

## What It Does

Charge-basis eigenbasis CZ gate simulator for Q-DTC-Q systems. Pre-computes E(φ) and non-adiabatic coupling M(φ) on a flux grid, interpolates with cubic splines, then evolves only 20–50 eigenstates via:

$$H_\text{eff}(t) = \text{diag}(E(\phi(t))) - i \cdot \dot\phi(t) \cdot M(\phi(t))$$

Two propagators:
- **LMDE**: Full dynamics with non-adiabatic coupling M (ground-truth leakage)
- **Trotter**: Adiabatic approximation (no M, zero leakage by construction)

Comparing LMDE vs Trotter cleanly isolates non-adiabatic effects.

---

## Package Structure

```
adiabatic_cz/
├── jax_setup.py      # configure_jax() — MUST call first
├── system.py         # ChargeTransmon, DTCSubsystem, DTCSystem
├── pulse.py          # ramen(), square_ramen(), PulseConfig
├── eigensolver.py    # compute_eigenbasis(), EigenbasisInterpolator
├── propagator.py     # propagate_lmde(), propagate_trotter()
├── metrics.py        # compute_all_metrics() → GateMetrics
├── simulation.py     # run_cz(), tune_amplitude(), run_amplitude_sweep()
├── plotting.py       # Plotly visualizations
├── math_utils.py     # cosm, sinm, hc helpers
└── io.py             # print/save results
```

---

## Quick Start

```python
from adiabatic_cz import configure_jax
configure_jax(num_cores=21)  # MUST be first

from adiabatic_cz import (
    DTCSystem, PulseConfig, RamenParams,
    run_cz, print_cz_result
)

# Big Endeavour parameters
simDict = {
    'transmons': {
        'Q0': {'f01': 4.5, 'anharm': 0.21, 'cutoff': 6},
        'DT1': {'f01': 3.8, 'anharm': 0.08, 'cutoff': 12},
        'Q1': {'f01': 5.0, 'anharm': 0.21, 'cutoff': 6}
    },
    'couplings': [
        ['Q0', 'DT1;0', 0.18],
        ['Q1', 'DT1;1', 0.18],
        ['DT1;0', 'DT1;1', 0.01],
    ],
}

system = DTCSystem.from_simdict(simDict, rm=0.25, n_dtc=12)

pulse = PulseConfig(
    gate_time_us=0.075,      # 75 ns
    amplitude=0.15,          # Phi_0 (toward anticrossing)
    pulse_type='square_ramen',
    ramen_width=0.020,       # 20 ns total rise+fall
)

result = run_cz(system, pulse, method='lmde', n_eigs=20)
print_cz_result(result)
```

---

## Key Workflows

### 1. Amplitude Sweep (explore parameter space)

```python
from adiabatic_cz import run_amplitude_sweep, plot_sweep_metrics

sweep = run_amplitude_sweep(
    system, gate_time_us=0.075,
    amp_range=(0.01, 0.22), n_points=40,
    method='lmde', n_eigs=20,
    pulse_type='square_ramen', ramen_width=0.020,
)
fig = plot_sweep_metrics(sweep)
fig.write_html('sweep.html')
```

### 2. Auto-Tune Amplitude (find CZ point)

```python
from adiabatic_cz import tune_amplitude

amp_opt, eig_data, metrics = tune_amplitude(
    system, gate_time_us=0.075,
    method='lmde', n_eigs=20,
    pulse_type='square_ramen', ramen_width=0.020,
    target_phase=np.pi/2,  # CZ gate
)
# Returns: amplitude, cached eigenbasis, metrics at CZ point
```

### 3. Fast Tuning via ZZ Integration (no ODE)

```python
from adiabatic_cz import tune_amplitude_zz, compute_zz_sweep

zz_data = compute_zz_sweep(system)
amp, _, phase = tune_amplitude_zz(
    system, gate_time_us=0.075,
    zz_data=zz_data,
    pulse_type='square_ramen', ramen_width=0.020,
    target_phase=np.pi,  # Full phase convention
)
```

### 4. Time-Resolved Dynamics

```python
from adiabatic_cz import run_dynamics, plot_population_dynamics

dyn = run_dynamics(system, pulse_config, method='lmde', n_times=500, eig_data=eig_data)
fig = plot_population_dynamics(dyn, eig_data=eig_data, system=system, input_idx=3)
fig.write_html('dynamics.html')
```

### 5. LMDE vs Trotter Comparison

```python
from adiabatic_cz import run_comparison, plot_comparison

comp = run_comparison(system, pulse_config, n_eigs=20)
fig = plot_comparison(comp)
# comp.delta_fidelity, comp.delta_leakage show non-adiabatic contribution
```

### 6. Eigenbasis Caching (reuse across pulses)

```python
from adiabatic_cz import compute_eigenbasis, save_eigenbasis, load_eigenbasis

eig_data = compute_eigenbasis(system, n_phi=501, n_eigs=20)
save_eigenbasis(eig_data, 'eigenbasis.npz')
# Later:
eig_data = load_eigenbasis('eigenbasis.npz')
# Pass eig_data= to any run_* function to skip recomputation
```

---

## Pulse Shapes

### Full Ramen
Smooth arch: idle → interaction → idle. Ramen math ensures crossing-aware angular velocity profile.
```python
PulseConfig(pulse_type='ramen', amplitude=0.15, gate_time_us=0.075)
```

### Square Ramen
Ramen-shaped rise/fall + flat top at interaction point. Best for studying ramp effects.
```python
PulseConfig(
    pulse_type='square_ramen',
    amplitude=0.15,
    gate_time_us=0.075,
    ramen_width=0.020,  # 20 ns total rise+fall (10 ns each)
)
```

**RamenParams** (default values work well):
- `det0=1.0` — detuning parameter
- `acceleration=2.0` — ramp aggressiveness (higher = sharper S-curve)
- `delta=10.0` — crossing gap parameter
- `target=0.5` — normalization (don't change)

---

## Key Data Classes

| Class | Fields | Description |
|-------|--------|-------------|
| `DTCSystem` | `.dims`, `.size`, `.phi_ext_0`, `.comp_indices` | Q-DTC-Q system |
| `PulseConfig` | `.gate_time_us`, `.amplitude`, `.pulse_type`, `.ramen_width` | Pulse definition |
| `EigenbasisData` | `.phi_grid`, `.eigenvalues`, `.eigenvectors`, `.coupling_matrix` | Pre-computed eigenbasis |
| `GateMetrics` | `.fidelity`, `.coherent_error`, `.avg_leakage`, `.per_state_leakage`, `.conditional_phase` | Gate quality |
| `CZResult` | `.metrics`, `.propagation`, `.eig_data`, `.pulse_config` | Full sim result |
| `SweepResult` | `.amplitudes`, `.metrics[]`, `.eig_data` | Amplitude sweep |
| `DynamicsResult` | `.tlist`, `.comp_populations`, `.total_leakage` | Time traces |

---

## Plotting Functions

| Function | Output |
|----------|--------|
| `plot_eigenbasis_spectrum(eig_data)` | Energy spectrum + ZZ vs flux (shared x-axis) |
| `plot_sweep_metrics(sweep)` | Gate error + leakage vs amplitude |
| `plot_population_dynamics(dyn, input_idx=3)` | Comp populations + leakage vs time |
| `plot_spectrum_pulse_overlay(eig_data, pc, phi0)` | Energy levels + pulse trajectory |
| `plot_interactive_leakage(dyn, ...)` | 3-panel interactive (spectrum + pulse + leakage) |
| `plot_coupling_matrix(eig_data)` | M matrix heatmap |
| `plot_comparison(comp)` | LMDE vs Trotter side-by-side |

All plots use bold black borders, plotly_white template, flux in Φ₀.

---

## Physics Notes

### Hierarchical Hamiltonian
1. Each transmon diagonalized in charge basis (exact, not Duffing)
2. DTC: two islands + shared junction → 36-dim → diag → truncate to 12 levels
3. Full system: 6 × 12 × 6 = 432 dimensions

### Non-Adiabatic Coupling
$$M_{ij}(\phi) = \langle \psi_i | \partial_\phi \psi_j \rangle$$
Computed via finite differences of phase-locked eigenvectors. Anti-Hermitian (M + M† = 0).

### Phase Convention
- Conditional phase π/2 = CZ gate (half-convention in metrics)
- ZZ integration gives full phase π (use `tune_amplitude_zz(target_phase=π)`)
- `tune_amplitude(target_phase=π/2)` for ODE-based tuning

### Flux Convention
- All in Φ_ext/Φ₀ (flux quanta)
- Positive amplitude pulses toward the anticrossing (toward lower |Φ|)
- Idle point: ~−0.33 Φ₀ for Big Endeavour

---

## Performance

| Operation | Time | Notes |
|-----------|------|-------|
| Eigenbasis (501 pts, 20 eigs) | ~3 s | 28 workers, 432-dim |
| Single LMDE gate | ~1 s | max_dt=0.5 µs |
| Single Trotter gate | ~0.2 s | 2000 steps |
| Amplitude sweep (40 pts, LMDE) | ~2 min | JIT compile: ~120 s first call |
| Amplitude sweep (40 pts, Trotter) | ~10 s | No JIT overhead |

### Tips
- `configure_jax(num_cores=21)` for LMDE (NUMA-safe on landsman2)
- `configure_jax(num_cores=28)` for Trotter (pure numpy, no JAX)
- Always pass `eig_data=` when running multiple pulses to avoid recomputation
- LMDE JIT compiles once per (gate_time, pulse_type, ramen_width); amplitude is traced

---

## Development Workflow

1. Edit locally: `~/repos/dtc_adiabatic_sim/`
2. `git commit && git push`
3. On landsman2: `cd ~/repos/dtc_adiabatic_sim && git pull`
4. Run: `python -u examples/script.py > examples/script.log 2>&1`
5. Results: `scp "landsman2:~/repos/dtc_adiabatic_sim/results/YYYYMMDD_*" ~/projects/dtc/sim_outputs/`

---

## Validated Results

| Configuration | Fidelity | Leakage | Phase |
|--------------|----------|---------|-------|
| Full Ramen 75ns | 0.9999 | 0.0004% | −0.507π |
| Square Ramen 75ns (rw=20ns) | TBD | TBD | TBD |
| LMDE vs Trotter match | ΔF < 1e-3 | — | — |

**Key validation**: LMDE with lab-frame H_eff (commit 43e9341) gives F=0.9999 for Full Ramen 75ns, matching expectations. Rotating-frame bug was fixed (must use lab-frame H_eff = diag(E) − i·dφ·M, not rotating frame without R†MR conjugation).
