# Rotating Frame Guide for CZ Gate Simulation

## Why the Rotating Frame is Needed

In the lab frame, each eigenstate accumulates phase at its eigenfrequency:
```
phase_j(t) = -2π * E_j * t    (thousands of radians for GHz × hundreds of ns)
```

The CZ conditional phase (~π) is a tiny difference of these huge numbers, making it:
- Hard to visualize in plots (diagonal phases span thousands of radians)
- Prone to numerical issues with `np.unwrap` (phase jumps > π per step)

The rotating frame removes the fast idle-frequency dynamics, revealing only the slow ZZ-driven CZ physics.

## Post-Hoc Transformation (Key Insight)

For piecewise-constant propagation, the rotating frame is a **post-hoc diagonal transformation** — the core propagation loop is unchanged:

```
U_rot_total = R(T_gate) @ U_lab_total      (since R(0) = I)
psi_rot(t_k) = R(t_k) @ psi_lab(t_k)
```

### Proof

At each step: `U_rot_k = R(t_{k+1}) @ U_lab_k @ R†(t_k)`

Total: `U_rot = U_rot_N ... U_rot_1 = R(t_N) @ U_lab_total @ R†(0) = R(T_gate) @ U_lab_total`

Since `R(0) = I`, we don't need to modify the propagation at all.

## Two Reference Options

### 1. `idle_eig` (Recommended)

Removes ALL phase accumulation at the idle flux point.

```python
R(t) = V_idle @ diag(exp(+i * 2π * E_idle * t)) @ V_idle†     (in bare basis)
```

- At idle: rotating-frame phases ≈ 0
- During pulse: phases show only flux-induced shifts relative to idle
- Cleanest for CZ gate plots

**Efficient shortcut** (no 432×432 matrix operations needed):
```python
# Computational subspace unitary
U_comp_rot[j, i] = exp(+i * 2π * E_idle_j * T_gate) * U_comp_lab[j, i]

# Population dynamics amplitudes
amp_rot_j(t) = exp(+i * 2π * E_idle_j * t) * amp_lab_j(t)
```

### 2. `bare`

Removes uncoupled (bare) transmon/DTC frequencies.

```python
R(t) = diag(exp(+i * 2π * H0_bare_diag * t))     (in bare basis)
```

- Leaves coupling-induced frequency shifts visible
- Useful for seeing hybridization effects
- No efficient shortcut — requires full 432×432 matrix multiply

## Frame-Independence Properties

For the `idle_eig` reference, since `R(t)` is diagonal in the eigenbasis:

```
<comp_j| R(t) |psi_lab> = exp(+i*2π*E_j*t) * <comp_j|psi_lab>
```

**Truly frame-independent** (proven, verified to <1e-15):
- **Populations**: `|<comp_j|psi_rot>|² = |<comp_j|psi_lab>|²`
- **Leakage**: `1 - Σ_j |U_comp[j,i]|²` (magnitudes unchanged)

**Differs by a known correction** (2π·T·ZZ_idle):
- **Conditional phase**: θ_rot = θ_lab + 2π·T·ZZ_idle (the rotating frame adds the idle ZZ contribution)
- **Fidelity**: changes correspondingly due to the phase shift in U_corrected

The correction vanishes when ZZ_idle → 0 (i.e. at the ideal idle point).

## Phase Tracking Gotcha

Lab-frame phases wrap rapidly (hundreds of radians per step). **Do NOT use** `np.unwrap(np.angle(...))` — it fails when the phase change per step exceeds π.

Instead, use incremental phase accumulation:
```python
# Safe for any phase accumulation rate
ratios = amp[1:] / amp[:-1]                    # always unit-magnitude
increments = np.angle(ratios)                   # always in (-π, π)
accumulated_phase = np.concatenate([[0], np.cumsum(increments)])
```

Handle near-zero amplitudes (leakage states):
```python
valid = np.abs(amp[:-1]) > 1e-15
ratios = np.ones(n_steps, dtype=complex)
ratios[valid] = amp[1:][valid] / amp[:-1][valid]
```

## API Usage

### Gate unitary with rotating frame
```python
from dtc_cz_sim.unitary_sim import compute_gate_unitary, extract_gate_metrics

result = compute_gate_unitary(
    system, phi_t, dt, phi_idle=phi_idle,
    rotating_frame='idle_eig',   # or 'bare'
)

# Lab frame metrics
metrics_lab = extract_gate_metrics(result['U_comp'])

# Rotating frame metrics (fidelity, conditional phase identical)
metrics_rot = extract_gate_metrics(result['U_comp_rot'])

# Rotating frame diagonal phases are ~0 at idle, ~π for |11>
print(np.angle(np.diag(result['U_comp_rot'])) / np.pi)
```

### Population dynamics with rotating frame
```python
from dtc_cz_sim.unitary_sim import population_dynamics

pop = population_dynamics(
    system, phi_t, dt, initial_state_label='11',
    phi_idle=phi_idle, rotating_frame='idle_eig',
)

# Lab populations (same as rotating — frame-independent)
pop['comp_populations']          # (n_steps+1, 4)

# Rotating frame phases (clean, near-zero at idle)
pop['comp_phases_rot']           # (n_steps+1, 4)

# Complex amplitudes
pop['comp_amplitudes']           # lab frame
pop['comp_amplitudes_rot']       # rotating frame
```

### Building the frame object directly
```python
from dtc_cz_sim.unitary_sim import build_rotating_frame

frame = build_rotating_frame(system, phi_idle, ref='idle_eig')
# frame['E_ref']   — reference energies (eigenvalues at idle)
# frame['V_idle']  — eigenvectors at idle
# frame['E_idle']  — idle eigenvalues
```

### Validation with sesolve (expm)
```python
from dtc_cz_sim.unitary_sim import propagate_sesolve, propagate_piecewise

U_eig = propagate_piecewise(system, phi_t, dt)['U_total']
U_expm = propagate_sesolve(system, phi_t, dt, method='expm')['U_total']

print(f"||U_eig - U_expm|| = {np.linalg.norm(U_eig - U_expm):.2e}")
# Typical: < 1e-10
```

## Demo Script

`examples/rotating_frame_demo.py` generates 5 plots:

1. **Phase comparison** (2×2): Lab (thousands of rad) vs rotating (near-zero)
2. **Conditional phase buildup** (2-panel): Lab vs rotating — identical shape, different scale
3. **Population dynamics** (2×2): Lab solid, rotating dotted — perfect overlap
4. **Gate unitary heatmaps** (2×2): Magnitude identical, phases differ dramatically
5. **Money plot**: All 4 diagonal phases in rotating frame; |11⟩ accumulates ~−π relative to others
