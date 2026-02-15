# Rotating Frame for CZ Gate Simulation

## Problem
Lab-frame simulation accumulates thousands of radians of phase (GHz frequencies x hundreds of ns gate time). The CZ conditional phase (~pi) is a tiny difference of these huge numbers — hard to visualize and interpret.

## Solution: Post-Hoc Rotating Frame

For piecewise-constant propagation, the rotating frame is simply:
```
U_rot_total = R(T_gate) @ U_lab_total
psi_rot(t_k) = R(t_k) @ psi_lab(t_k)
```
where R(t) = diag(exp(+i * 2pi * E_ref * t)) in the appropriate basis.

**No changes to the core propagation loop needed.**

### Proof
At each step: U_rot_k = R(t_{k+1}) @ U_lab_k @ R†(t_k)
Total: U_rot = U_rot_N ... U_rot_1 = R(t_N) @ U_lab_total @ R†(0) = R(T_gate) @ U_lab_total

## Two Reference Frame Options

### 1. Bare diagonal (`ref='bare'`)
- R(t) = diag(exp(+i * 2pi * H0_bare_diag * t)) in bare basis
- Removes uncoupled transmon/DTC frequencies
- Leaves coupling-induced shifts visible
- H0_bare is diagonal in bare basis by construction (sum of Kronecker products of diagonal element Hamiltonians)

### 2. Idle eigenvalues (`ref='idle_eig'`) — RECOMMENDED
- R(t) = V_idle @ diag(exp(+i * 2pi * E_idle * t)) @ V_idle† in bare basis
- Removes ALL phase accumulation at idle
- At idle: phases ~ 0. During pulse: shows only flux-induced shift
- Cleanest for CZ plots

## Frame-Independence Properties

For `idle_eig` reference:
- **Populations**: |<comp_j|psi_rot>|² = |<comp_j|psi_lab>|² (R is diagonal in eigenbasis)
- **Gate fidelity**: identical in both frames
- **Conditional phase**: identical in both frames
- **Leakage**: identical in both frames

Proof: <comp_j|R(t)|psi_lab> = exp(+i*2pi*E_j*t) * <comp_j|psi_lab>, so |amplitude|² unchanged.

## Efficient Computation

For `idle_eig`, the 4x4 rotating-frame gate unitary has a shortcut:
```
U_comp_rot[j,i] = exp(+i*2pi*E_idle_j*T_gate) * U_comp_lab[j,i]
```
No need to transform the full 432x432 matrix.

For population dynamics, rotating-frame amplitudes:
```
amp_rot_j(t) = exp(+i*2pi*E_idle_j*t) * amp_lab_j(t)
```
No matrix multiply needed.

## Phase Tracking Gotcha

Lab-frame phases wrap rapidly (hundreds of radians per step). Do NOT use `np.unwrap(np.angle(...))` — it fails when phase change per step exceeds pi.

Instead, use incremental phase accumulation:
```python
phase_increments = np.angle(amp[1:] / amp[:-1])  # always in (-pi, pi)
accumulated_phase = np.concatenate([[0], np.cumsum(phase_increments)])
```

## Implementation Plan

See `~/.claude/plans/cuddly-sleeping-newell.md` for the full implementation plan.

Files to modify:
1. `dtc_hamiltonian.py` — add `get_bare_diagonal(phi)` method
2. `unitary_sim.py` — add `build_rotating_frame()`, modify `compute_gate_unitary()` and `population_dynamics()`, add `propagate_sesolve()`
3. New `examples/rotating_frame_demo.py` — 5 comparison plots

## sesolve Note

sesolve does NOT speed up the simulation for piecewise-constant H (eigendecomposition is exact per step). But it provides:
1. Validation via different numerical path (scipy.expm)
2. Groundwork for future Lindblad/mesolve extension
