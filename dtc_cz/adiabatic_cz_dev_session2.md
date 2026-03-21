# adiabatic_cz Session 2: Hierarchical Hamiltonian & Flux Quanta (Feb 25, 2026)

## What Was Done

### 1. Committed & pushed scipy eigensolver fix (`c125c2d`)
- Sequential `scipy.linalg.eigh` loop (replacing vmap)
- `cosm_np`/`sinm_np` (scipy) and `build_H0(use_jax=False)`

### 2. Hierarchical Hamiltonian Construction (`34d7733`)
**New class: `DTCSubsystem`** — the core of the hierarchical approach:
- CL × CR tensor product (6×6 = 36 dim) + shared junction cosine
- Diagonalize at idle flux, truncate to 12 levels
- Store truncated ladder operators for qubit-coupler exchange coupling

**Refactored `DTCSystem`** to use `DTCSubsystem`:
- Old: Q0 × Q1 × CL × CR = 6×6×6×6 = **1296 dim**, cosm on 1296×1296
- New: Q0 × DTC × Q1 = 6×12×6 = **432 dim**, cosm on 36×36
- `build_H0(phi)`: builds DTC H at 36-dim, projects to 12-dim, embeds in 432-dim

### 3. Complex eigenvector dtype fix (`ef001da`)
- Hierarchical projection gives complex-typed H0 (though imaginary part = 0)
- Allocated `all_evecs` as complex to avoid silent truncation

### 4. Real eigh optimization (`5a8a7fd`)
- Discovered: `scipy-openblas64` (ILP64) makes complex `eigh` on 432-dim take **11.8s** (vs 48ms for real)
- H0 is effectively real (max imag = 0), so cast to real before `eigh`
- Also: 21 cores optimal on landsman3 (28 cores hits NUMA overhead)

### 5. Adiabatic state tracking (`e34b906`, `d7c4a43`)
- Replaced diagonal-only phase locking with full overlap matrix tracking
- `_adiabatic_track()`: greedy argmax with exclusion, starting from idle
- Tracks outward in both directions from idle (where states are most pure)
- Fixed default range to center on idle (not hardcoded [π, 2π])

### 6. Flux quanta conversion (`e93d677`)
Converted all external-facing flux parameters from radians to Φ_ext/Φ_0:
- `system.phi_ext_0` now in flux quanta (-0.3304 for rm=0.25)
- `PulseConfig.amplitude` in flux quanta (was `amplitude_rad` in radians)
- Eigensolver `phi_grid` in flux quanta
- Default eigensolver range: [-0.5, idle+0.08] Phi_0 (asymmetric, most below idle)
- Internal cosine uses `2π * phi_ext` for correct phase argument
- All docstrings updated

## Test Results (landsman3, 21 cores, 501 pts)

| Test | Result |
|------|--------|
| System construction | dims=[6,12,6]=432 |
| phi_ext_0 | -0.3304 Phi_0 |
| H0 build time | **3.1 ms** |
| Eigensolver (501pt, 20eigs) | **19.6s** |
| ZZ at idle | -0.040 MHz |
| Comp state eigenbasis indices | [0, 4, 3, 13] |
| Adiabatic tracking | min overlap = 0.799 (501pts) |
| Trotter (A=-0.15) | F=0.644, leak=0%, phase=1.035π |
| LMDE (A=-0.15) | F=0.560, leak=7.3%, phase=0.995π |
| LMDE unitarity | max|UU†-I| = 3.1e-5 |

## Performance Summary

| Operation | Old (flat 1296) | New (hierarchical 432) |
|-----------|----------------|----------------------|
| H0 build (scipy) | ~550 ms | **3 ms** (180x faster) |
| Eigensolver (501 pts) | hung >10min | **19.6s** |
| Hilbert space dim | 1296 | 432 |
| cosm matrix dim | 1296 | 36 |

## Current Issues

### Adiabatic tracking still imperfect near sweet spot
Min consecutive overlap = 0.799 at 501 points. Near -0.5 Phi_0 (sweet spot)
there are strong avoided crossings that challenge the greedy matching.
This may need:
- Denser grid near crossings
- Or simply avoid sweeping all the way to the sweet spot

## Bugs Found

### scipy-openblas64 complex eigh pathologically slow
- `scipy.linalg.eigh` on 432-dim complex: **11.8s** with 28 cores
- Root cause: scipy's bundled `scipy-openblas64` (ILP64 interface)
- Real symmetric same size: 48ms
- Fix: cast H to real (imaginary part is negligible)
- Also: use 21 cores (avoids NUMA overhead on landsman3)

## Git Status
- 9 commits on master, all pushed
- Latest: `e93d677` (flux quanta conversion)

## Next Steps

1. **Improve tracking near sweet spot** — denser grid or adaptive grid near crossings
2. **Tune pulse amplitude** for π conditional phase
3. **Cross-validate with SmarterThanARock** at idle point
4. **CCC GPU testing**
