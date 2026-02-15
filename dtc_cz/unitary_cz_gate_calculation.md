# How the Unitary CZ Gate is Calculated

## Overview

The CZ gate simulation has two layers:
- **Layer 0 (Adiabatic)**: Integrates ZZ(φ(t)) along the pulse — fast but ignores leakage
- **Layer 1 (Unitary)**: Full Schrodinger evolution in the 432-dim Hilbert space — captures leakage and non-adiabatic effects

This document describes Layer 1 in detail.

## Two Bases

The 432-dimensional Hilbert space (Q0⊗DTC⊗Q1 = 6×12×6) has two natural bases:

### Bare basis (tensor product, fixed)
Product states `|n_Q0, n_DTC, n_Q1>`, independent of flux. Ordered as:
```
Index 0:  |0, 0, 0>
Index 1:  |0, 0, 1>
...
Index 5:  |0, 0, 5>
Index 6:  |0, 1, 0>
...
Index 72: |1, 0, 0>
Index 73: |1, 0, 1>
...
```
General formula: `index = n_Q0 * 72 + n_DTC * 6 + n_Q1`

### Eigenbasis at flux φ (energy-sorted, flux-dependent)
Eigenstates of H(φ), sorted by energy. At each flux point, diagonalization gives:
```
H(φ) = V(φ) @ diag(E(φ)) @ V(φ)†
```
- `E(φ)`: eigenvalues (energies in GHz), sorted ascending
- `V(φ)`: eigenvectors — column k is the k-th eigenstate in the bare basis
- `V(φ)[:, k]` is a 432-element vector giving the bare-basis decomposition of eigenstate k

## Step 1: Full Unitary Propagation

The flux pulse φ(t) is discretized into N steps of duration dt.

At each step k:
```
U_step_k = exp(-i · 2π · H(φ_k) · dt)
         = V_k @ diag(exp(-i · 2π · E_k · dt)) @ V_k†
```

This is **exact** for constant H over dt (not an approximation).

The total propagator (in **bare basis**):
```
U_total = U_step_N @ U_step_{N-1} @ ... @ U_step_1
```

U_total is a 432×432 unitary matrix mapping bare-basis input to bare-basis output.

## Step 2: Identify Computational States at Idle

At the idle flux φ_idle, diagonalize to get V_idle and find which eigenstates
correspond to the 4 computational states:

```
get_computational_states(φ_idle) returns:
  |00> → eigenstate index 0  → V_idle[:, 0]  (mostly |0,0,0> in bare basis)
  |01> → eigenstate index 3  → V_idle[:, 3]  (mostly |0,0,1> in bare basis)
  |10> → eigenstate index 2  → V_idle[:, 2]  (mostly |1,0,0> in bare basis)
  |11> → eigenstate index ~5 → V_idle[:, 5]  (mostly |1,0,1> in bare basis)
```

The eigenstate index ≠ the bare basis index! This was the critical bug we fixed.

## Step 3: Extract 4×4 Gate Unitary

For each computational input state i (i = 00, 01, 10, 11):

```python
# Express eigenstate i in bare basis
psi_in = V_idle[:, comp_idx_i]      # 432-element vector

# Propagate through the gate
psi_out = U_total @ psi_in          # 432-element vector

# Project onto computational output states
for j in [00, 01, 10, 11]:
    U_comp[j, i] = V_idle[:, comp_idx_j].conj() @ psi_out
```

This gives a 4×4 complex matrix U_comp.

### Why project back onto V_idle eigenstates?

The gate starts and ends at φ_idle. The computational states are defined as
eigenstates at idle. So the natural input/output basis is the eigenbasis at
φ_idle, not the bare basis.

### Leakage

For input state i:
```
Leakage_i = 1 - Σ_j |U_comp[j, i]|²
```
i.e., the probability that leaks OUT of the 4-dimensional computational subspace.

## Step 4: Extract Gate Metrics

From U_comp, extract diagonal phases:
```
θ_ij = arg(U_comp[ij, ij])    for ij ∈ {00, 01, 10, 11}
```

### Conditional phase (CZ condition)
```
θ_cond = θ_11 - θ_10 - θ_01 + θ_00
```
For a perfect CZ gate: θ_cond = ±π

### Single-qubit Z corrections
```
Z_Q0 = θ_10 - θ_00    (phase accumulated on Q0)
Z_Q1 = θ_01 - θ_00    (phase accumulated on Q1)
```
These are absorbed into virtual Z gates in the quantum circuit.

### Corrected unitary
```
U_corrected = exp(-iθ_00) · diag(1, e^{-iZ_Q1}, e^{-iZ_Q0}, e^{-i(Z_Q0+Z_Q1)}) @ U_comp
```

### Average gate fidelity
```
M = U_ideal† @ U_corrected
F = (|Tr(M)|² + d) / (d² + d)    where d = 4
```

U_ideal = diag(1, 1, 1, -1) for CZ.

## The Bug We Fixed (2025-02-14)

The original code used eigenstate indices as bare-basis indices:
```python
# WRONG: eigenstate index ≠ bare basis index
psi_out = U_total[:, comp_idx]     # column comp_idx of U_total = bare vector comp_idx
U_comp[j,i] = psi_out[comp_idx_j]  # element comp_idx_j of psi_out
```

For |01> (eigenstate #3), this propagated bare vector #3 = |0,0,3> (Q1 third level),
not |0,0,1> (Q1 first excitation). Result: ~40% fidelity, ~40% leakage.

Fixed by using V_idle to transform between bases:
```python
# CORRECT: use eigenvectors to transform
psi_in = V_idle[:, comp_idx_i]
psi_out = U_total @ psi_in
U_comp[j,i] = V_idle[:, comp_idx_j].conj() @ psi_out
```

## Rotating Frame

Lab-frame phases accumulate thousands of radians (GHz × hundreds of ns), burying the CZ conditional phase (~π). The rotating frame removes this fast dynamics via a post-hoc transformation:

```
U_rot_total = R(T_gate) @ U_lab_total
```

Two references available: `idle_eig` (recommended — removes all idle eigenvalues) and `bare` (removes uncoupled frequencies). Populations, fidelity, and conditional phase are frame-independent.

See `rotating_frame_guide.md` for full details, proofs, API examples, and the phase tracking gotcha.
