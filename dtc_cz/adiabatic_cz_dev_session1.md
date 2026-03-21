# adiabatic_cz Session 1: Initial Implementation & Testing (Feb 25, 2026)

## What Was Built

All 15 files implemented and committed (2 commits on master, pushed to `git@github.ibm.com:Ruichen-Zhao/dtc_adiabatic_sim.git`).

**Package** (`adiabatic_cz/`):
- `__init__.py` — lazy imports
- `jax_setup.py` — `configure_jax()` (from duffing_cz)
- `math_utils.py` — `hc, freq, cosm, sinm, cosm_np, sinm_np`
- `system.py` — **NEW**: `ChargeTransmon` (charge-basis diag) + `DTCSystem` with `from_simdict()`
- `pulse.py` — `PulseConfig`, `pulse_with_deriv()` via `jax.grad`
- `eigensolver.py` — **NEW CORE**: `compute_eigenbasis()`, `EigenbasisInterpolator` (interpax)
- `propagator.py` — **NEW CORE**: `propagate_lmde()` + `propagate_trotter()`
- `simulation.py` — `run_cz()`, `run_comparison()`, `run_dynamics()`
- `metrics.py` — from duffing_cz
- `plotting.py` — `plot_eigenbasis_spectrum`, `plot_coupling_matrix`, `plot_comparison`, `plot_spline_quality`
- `io.py` — `print_cz_result`, `print_comparison`, `save_cz_result`

**Examples**: `run_basic_cz.py`, `compare_lmde_trotter.py`, `eigenbasis_diagnostics.py`

## Bugs Found & Fixed

### 1. H_eff missing `-i` factor (committed `fa1aefb`)
- Adiabatic Schrodinger: `i*dc/dt = E*c - i*(dphi/dt)*M*c`
- Correct: `H_eff = diag(E) - i*dphi/dt*M`
- Was: `H_eff = diag(E) + dphi/dt*M` (non-Hermitian, non-unitary evolution)
- Fix gives max|UU†-I| = 1.67e-14

### 2. vmap eigensolver too slow (uncommitted fix)
- `jax.vmap(jnp.linalg.eigh)` over 101 × 1296-dim: 60 GB RAM, >10 min compile on landsman3
- Switched to sequential `scipy.linalg.eigh` loop
- Added `cosm_np`/`sinm_np` (scipy) and `build_H0(use_jax=False)`

## Test Results (local, small grids)

| Test | Result |
|------|--------|
| System construction | dims=[6,6,6,6]=1296 |
| Eigensolver (21pt, 20eigs) | ZZ = -0.045 MHz (reasonable for DTC) |
| Comp state mapping | [0, 4, 3, 13] verified vs direct diag |
| Interpolation at grid | Error = 0 (exact) |
| Phase locking | min overlap > 0.99 |
| Coupling matrix | Real anti-symmetric (verified) |
| Trotter | F=0.851, leak=0%, phase=0.22π |
| LMDE | F=0.634, leak=11.5%, phase=0.10π, unitary |
| Pulse autodiff | dphi/dt correct (0 at peak, ±53 at edges) |
| run_cz API | End-to-end works |

## Critical Issue: Hilbert Space Too Large

Current: 4 individual transmons → 6×6×6×6 = **1296 dim**. Each `build_H0` calls `cosm()` on 1296×1296 matrix.

### Seth's Hierarchical Approach (from `cauer_package`)
Location: `~/repos/research-novel-devices/cauer/quantum_model.py`

Key class: `hierarchical_charge_hami` with `tensor_and_diagonalize()`:
1. Diagonalize each junction individually (30-35 charge states → 5-6 levels)
2. **Combine DTC islands first**: CL×CR (6×6=36 dim) + shared junction → diag → truncate to 12 levels
3. Build full system: Q0×DTC×Q1 = 6×12×6 = **432 dim**
4. `cosm()` only on 36-dim subspace (fast!)

Usage: `ch.solve([ej1, ej2], n_qubit=6, n_diag=35, reductions=[reduction([0,1], 15)])`

This matches SmarterThanARock's approach. **Must implement this.**

## Next Steps (Priority Order)

1. **Hierarchical Hamiltonian** — `DTCSubsystem` class, reduce 1296 → 432 dim
2. **Commit & push** scipy eigensolver fix
3. **Test on landsman3** with full grid (201-501 points)
4. **Tune pulse amplitude** for π phase in charge-basis model
5. **Cross-validate** with SmarterThanARock at idle
6. **CCC GPU** testing

## Git Status
- Repo: `git@github.ibm.com:Ruichen-Zhao/dtc_adiabatic_sim.git`
- Local: `~/repos/dtc_adiabatic_sim/`
- landsman3: `/home/US8J4928/repos/dtc_adiabatic_sim/` (cloned, installed)
- Commits: `f021123` (initial), `fa1aefb` (H_eff fix)
- Uncommitted: scipy eigensolver fix

## Seth's cauer_package Reference
- Location: `~/repos/research-novel-devices/cauer/`
- `quantum_model.py` — `hierarchical_charge_hami`, `charge_hami.junction_hami()`, `tensor_and_diagonalize()`
- `qm_flex.py` — `hierarchical_flex_hami` with `resolve()` (re-solve at different flux without re-building basis)
- `en2charge.py` — 2-junction example with `reduction([0,1], 15)`
- Uses: numpy, scipy, qutip. No JAX.
