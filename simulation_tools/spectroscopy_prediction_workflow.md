# Technical Details: Spectroscopy Prediction Workflow

> **See also:** [[Quantum Simulation Concepts]], [[Coupling and Transition Matrix Visualization]], [[DTC State Labeling]], [[Parallel Simulation Benchmarks]]

This document explains the complete simulation workflow from Hamiltonian construction to spectroscopy map generation.

---

## Overview: Simulation Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SPECTROSCOPY PREDICTION FLOW                         │
└─────────────────────────────────────────────────────────────────────────────┘

FOR EACH FLUX POINT φ:

┌──────────────────┐
│  1. simDict      │  Device parameters: f01, anharm, couplings, cutoffs
│     + φ value    │  
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. simOne()     │  Build H_total (3888×3888), diagonalize
│                  │  → eigenvalues E_k, eigenvectors V
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. Build Q_bare │  Q_R0 = (a+a†)_R0 ⊗ I ⊗ I ⊗ I ⊗ I  (3888×3888)
│                  │  Resonator drive operator in bare product basis
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. M = V† Q V   │  Transform to dressed basis (3888×3888)
│                  │  M[i,j] = ⟨ψ_i|Q_R0|ψ_j⟩
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. Extract      │  strengths[k] = |M[0,k]|² (ground state row)
│     |M|²         │  freqs[k] = E_k - E_0 (transition frequencies)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  6. Filter &     │  Keep: freq_min < f < freq_max
│     Store        │        |M|² > threshold
└──────────────────┘

AFTER ALL FLUX POINTS:

┌──────────────────┐
│  7. Plot         │  x = φ, y = frequency, color = log₁₀(|M|²)
│     2D Map       │  
└──────────────────┘
```

---

## Key Matrices and Vectors

### Summary Table

| Name | Dimensions | Basis | Description |
|------|------------|-------|-------------|
| `H_total` | 3888 × 3888 | Bare product | Full system Hamiltonian |
| `V_full` | 3888 × 3888 | Columns = dressed, Rows = bare | Eigenvector matrix |
| `V_full[:, k]` | 3888 × 1 | Bare product | Single eigenvector \|ψ_k⟩ |
| `Q_bare` | 3888 × 3888 | Bare product | Resonator (a+a†) in full Hilbert space |
| `M_dressed` | 3888 × 3888 | Dressed | Matrix elements ⟨ψ_i\|Q\|ψ_j⟩ |
| `eigenvalues` | 3888 | - | Energy levels E_k (sorted) |

### V_full: The Eigenvector Matrix

**V_full is a matrix, NOT a vector.** Each column is one eigenvector.

```
V_full = 3888 × 3888 matrix

         │  |ψ_0⟩   |ψ_1⟩   |ψ_2⟩  ...  |ψ_3887⟩  │
         │   ↓       ↓       ↓            ↓       │
         ┌─────────────────────────────────────────┐
|bare_0⟩ │  v_00    v_01    v_02   ...   v_0,3887 │  ← row 0
|bare_1⟩ │  v_10    v_11    v_12   ...   v_1,3887 │  ← row 1
|bare_2⟩ │  v_20    v_21    v_22   ...   v_2,3887 │  ← row 2
   ⋮     │   ⋮       ⋮       ⋮      ⋱      ⋮      │
|bare_3887⟩│ v_3887,0  ...              v_3887,3887│  ← row 3887
         └─────────────────────────────────────────┘
              ↑
           Column k = eigenvector |ψ_k⟩ 
                      expressed in bare basis
```

**Index meanings:**
- **Column k**: The k-th dressed eigenstate |ψ_k⟩ (sorted by energy, k=0 is ground)
- **Row i**: Coefficient of bare basis state |bare_i⟩
- **V[i, k]**: ⟨bare_i|ψ_k⟩ = amplitude of bare state i in dressed state k

**Mathematical relation:**

$$|\psi_k\rangle = \sum_i V_{ik} |bare_i\rangle$$

### M_dressed: The Transition Matrix

**M_dressed is NOT the Hamiltonian!** It is the resonator coupling operator (a+a†) in the dressed basis.

```
M_dressed[i,j] = ⟨ψ_i| (a + a†)_R0 |ψ_j⟩

         |ψ_0⟩   |ψ_1⟩   |ψ_2⟩   |ψ_3⟩  ...
        ┌──────┬──────┬──────┬──────┬─────
|ψ_0⟩   │  ~0  │ M_01 │ M_02 │ M_03 │ ...    ← Ground state row
        ├──────┼──────┼──────┼──────┼─────
|ψ_1⟩   │ M_10 │  ~0  │ M_12 │ M_13 │ ...
        ├──────┼──────┼──────┼──────┼─────
|ψ_2⟩   │ M_20 │ M_21 │  ~0  │ M_23 │ ...
        ├──────┼──────┼──────┼──────┼─────
  ⋮     │  ⋮   │  ⋮   │  ⋮   │  ⋮   │ ⋱
```

| Element | Formula | Meaning |
|---------|---------|---------|
| **Diagonal M[k,k]** | ⟨ψ_k\|(a+a†)\|ψ_k⟩ ≈ 0 | Expectation value (usually ~0) |
| **Off-diagonal M[i,j]** | ⟨ψ_i\|(a+a†)\|ψ_j⟩ | Transition amplitude between states |
| **\|M[0,k]\|²** | \|⟨ψ_k\|Q\|ψ_0⟩\|² | **Spectroscopy strength** for 0→k |

---

## Detailed Code Walkthrough

### Step 1: Define System Parameters

```python
simDict = {
    "transmons": {
        "Q0":  {"f01": 4.5, "anharm": 0.21, "cutoff": 6},
        "DT1": {"f01": 3.8, "anharm": 0.08, "g": 0.7, "phi": φ, "cutoff": 12},
        "Q1":  {"f01": 5.0, "anharm": 0.21, "cutoff": 6}
    },
    "resonators": {
        "R0": {"f_r": 7.04, "n_levels": 3},
        "R1": {"f_r": 7.2,  "n_levels": 3}
    },
    "couplings": [
        ["Q0", "DT1;0", 0.18],
        ["Q1", "DT1;1", 0.18],
        ...
    ]
}
```

### Step 2: Loop Over Flux Points

```python
phi_values = np.linspace(0.0, 0.5, 101)  # 101 flux points

for phi in phi_values:
    # Update flux in simDict
    dict_setter(sd, ["transmons", "DT1", "phi"], phi)
    
    # Full diagonalization
    result = simOne(sd, return_full=True)
```

**result dictionary contains:**

| Key | Shape | Description |
|-----|-------|-------------|
| `eigenvalues` | (3888,) | Sorted energies E_k |
| `eigenvectors` | (3888, 3888) | V matrix, columns = dressed states |
| `resonator_Q_ops` | dict | `{"R0": Q_R0_bare, "R1": Q_R1_bare}` |
| `state_labels_ordered` | list[3888] | Labels like "00000", "10000", ... |

### Step 3: How Q_bare is Constructed

Inside `simOne()`, the resonator coupling operator is built via Kronecker products:

```python
# Element order: [Q0, DT1, Q1, R0, R1]
# Dimensions:      6  ×  12  ×  6  ×  3  ×  3  = 3888

# For resonator R0:
Q_list = [I_Q0,   I_DT1,  I_Q1,  Q_R0,  I_R1]
#         6×6    12×12    6×6    3×3    3×3

Q_R0_bare = kron_it(Q_list)  # = I ⊗ I ⊗ I ⊗ Q_R0 ⊗ I
# Result: 3888 × 3888 matrix
```

Where `Q_R0` is the local resonator operator:

```python
# Lowering operator for 3-level resonator
a = [[0, 1,   0  ],
     [0, 0,   √2 ],
     [0, 0,   0  ]]

Q_R0 = a + a†  =  [[0, 1,   0  ],
                   [1, 0,   √2 ],
                   [0, √2,  0  ]]   # 3 × 3, Fock basis
```

### Step 4: Compute Matrix Elements

```python
M_sq, labels = get_transition_matrix_elements(result, "R0")
```

Inside this function:

```python
def get_transition_matrix_elements(full_result, resonator_name="R0"):
    V = full_result["eigenvectors"]              # 3888 × 3888
    Q_bare = full_result["resonator_Q_ops"][resonator_name]  # 3888 × 3888

    # Transform Q to dressed basis: M = V† Q V
    M_complex = V.conj().T @ Q_bare @ V          # 3888 × 3888

    # Square to get transition probabilities
    M_squared = np.abs(M_complex)**2             # |⟨ψ_i|Q|ψ_j⟩|²

    return M_squared, labels
```

### Step 5: Extract Ground State Transitions

```python
E = result["eigenvalues"]       # [E_0, E_1, E_2, ...]
ground_idx = 0                  # Lowest energy state

# Transition frequencies from ground
freqs = E - E[ground_idx]       # [0, E_1-E_0, E_2-E_0, ...]

# Transition strengths from ground (row 0 of M_squared)
strengths = M_sq[ground_idx, :] # |⟨ψ_k|Q_R0|ψ_0⟩|² for all k
```

### Step 6: Filter and Store

```python
mask = (
    (freqs >= 0.5) &              # Frequency window
    (freqs <= 15.0) &
    (strengths >= 1e-10) &        # Strength threshold
    (np.arange(len(freqs)) != 0)  # Exclude ground→ground
)

point_data = {
    "freqs": freqs[mask],
    "strengths": strengths[mask],
    "labels": [labels[j] for j in np.where(mask)[0]],
    ...
}
```

### Step 7: Plot 2D Spectroscopy Map

```python
# Collect all points across all flux values
for i, phi in enumerate(phi_values):
    td = spec_data["transitions"][i]
    phi_all.extend([phi] * len(td["freqs"]))
    freq_all.extend(td["freqs"])
    strength_all.extend(td["strengths"])

# Scatter plot with log color scale
color = np.log10(strength_all)
plt.scatter(phi_all, freq_all, c=color, cmap="inferno")
```

---

## Physical Interpretation

### What M² Tells You

| Value | Meaning |
|-------|---------|
| **Large \|M[0,k]\|²** | Strong transition, **bright** spectroscopy line |
| **Small \|M[0,k]\|²** | Weak transition, **dim** line |
| **\|M[0,k]\|² ≈ 0** | **Dark** transition, invisible via this drive port |

### Sum Rule Validation

```python
sum_rule = np.sum(strengths)  # Should be ≈ 1.0
```

This checks completeness: the ground state must transition to *somewhere* with total probability 1.

### Dual-Drive Phase Control

For phase-controlled dual-port drive:

$$Q_{eff}(\delta) = Q_{R0} + e^{i\delta} Q_{R1}$$

$$|M(\delta)|^2 = |\langle \psi_k | Q_{R0} + e^{i\delta} Q_{R1} | \psi_0 \rangle|^2$$

- **δ = 0**: Symmetric combination → enhances N-like DTC modes
- **δ = π**: Antisymmetric combination → enhances T-like DTC modes

---

## Basis Conventions

### Bare Product Basis

States are labeled by occupation numbers: `|n_Q0, n_DT1, n_Q1, n_R0, n_R1⟩`

Example labels:
- `"00000"` = ground state (all subsystems in |0⟩)
- `"10000"` = Q0 excited
- `"01000"` = DTC in |1⟩
- `"00100"` = Q1 excited
- `"00010"` = R0 has 1 photon

### Dressed Eigenbasis

After diagonalization, states are labeled by energy order:
- |ψ_0⟩ = ground state (lowest energy)
- |ψ_1⟩ = first excited state
- etc.

Each dressed state is a superposition of bare states:

$$|\psi_k\rangle = \sum_i V_{ik} |bare_i\rangle$$

**WARNING:** The state label (e.g., "01000") shows only the **dominant** bare component, not the full wavefunction composition. Hybridized states can have significant contributions from multiple bare states.

---

## Key Formulas

| Quantity | Formula | Code |
|----------|---------|------|
| Transition frequency | $f_k = E_k - E_0$ | `freqs = E - E[0]` |
| Transition strength | $\|M_{0k}\|^2 = \|\langle\psi_k\|Q\|\psi_0\rangle\|^2$ | `strengths = M_sq[0, :]` |
| Matrix element | $M = V^\dagger Q V$ | `M = V.conj().T @ Q @ V` |
| Sum rule | $\sum_k \|M_{0k}\|^2 = 1$ | `np.sum(strengths)` |

---

## File Structure

| File | Purpose |
|------|---------|
| `spectroscopy_prediction.py` | Core computation: flux sweep, matrix elements |
| `dual_drive_spectroscopy.py` | Phase-controlled dual-port analysis |
| `run_spectroscopy.py` | Main entry point for basic sweeps |
| `run_phase_modulation.py` | Single-flux phase sweep |
| `run_dual_drive_comparison.py` | δ=0 vs δ=π comparison |
| `plot_lines_v2.py` | Continuous line visualization |
| `plot_interactive.py` | Plotly interactive viewer |

---

## Typical Parameters (Big Endeavour)

```python
simDict = {
    "transmons": {
        "Q0":  {"f01": 4.5, "anharm": 0.21, "cutoff": 6},
        "DT1": {"f01": 3.8, "anharm": 0.08, "g": 0.7, "phi": 0.4, "cutoff": 12},
        "Q1":  {"f01": 5.0, "anharm": 0.21, "cutoff": 6}
    },
    "resonators": {
        "R0": {"f_r": 7.04, "n_levels": 3},
        "R1": {"f_r": 7.2,  "n_levels": 3}
    },
    "couplings": [
        ["Q0", "DT1;0", 0.18],
        ["Q1", "DT1;1", 0.18],
        ["DT1;0", "DT1;1", 0.01],
        ["R0", "Q0", 0.13],
        ["R1", "Q1", 0.13],
        ["R0", "DT1;0", 0.03],
        ["R1", "DT1;1", 0.03],
    ],
    "N": 12,
    "cutoff": 5
}
```

**Total Hilbert space: 6 × 12 × 6 × 3 × 3 = 3,888 dimensions**
