# Technical Details: Coupling & Transition Matrix Visualization

> **See also:** [[Quantum Simulation Concepts]], [[Spectroscopy Prediction Workflow]], [[DTC State Labeling]]

This document explains the simulation workflow and compares it with the spectroscopy prediction tool.

---

## Overview: What This Tool Computes

This tool computes matrix elements of operators in the **dressed eigenbasis** of a Q-DTC-Q system:

1. **Coupling Matrix**: |⟨ψ_i|H_c|ψ_j⟩| — inter-element coupling Hamiltonian
2. **Transition Matrix**: |⟨ψ_i|Q_qubit|ψ_j⟩| — qubit charge operator

Both are computed via the same transformation: **M = V† O V**, where:
- V = eigenvector matrix (columns = dressed states)
- O = operator in bare product basis (H_c or Q_qubit)

---

## Key Matrices and Operators

### Summary Table

| Name | Dimensions | Description |
|------|------------|-------------|
| `H_bare` | 432 × 432 | Uncoupled Hamiltonian (sum of individual H) |
| `H_c` | 432 × 432 | Inter-element coupling Hamiltonian |
| `H_total` | 432 × 432 | H_bare + H_c |
| `V` | 432 × 432 | Eigenvector matrix (columns = dressed states) |
| `Q_Q0_full` | 432 × 432 | Q0 charge operator in full Hilbert space |
| `Q_Q1_full` | 432 × 432 | Q1 charge operator in full Hilbert space |
| `M_coupling` | N × N | H_c in dressed basis (truncated to N levels) |
| `M_transition` | N × N | Q_qubit in dressed basis (truncated to N levels) |

---

## Detailed Workflow

### Step 1: Build Individual Subsystem Operators

```python
for transmon in ['Q0', 'DT1', 'Q1']:
    if 'DT' in transmon:
        H, Q, I, N = get_single_DT(Ej, Ec, ng, g, phi, Z, ...)
        # Q is a list: [Q1, Q2, Q1*Q2] for DTC
    else:
        H, Q, I, N = get_single_qubit(Ej, Ec, ng, ...)
        # Q is the charge operator in energy eigenbasis

    op_dict[transmon] = {'H': H, 'Q': Q, 'I': I, 'N': N, 'Z': Z}
```

Dimensions:
- Q0: 6 × 6
- DT1: 12 × 12
- Q1: 6 × 6

### Step 2: Build Full Hilbert Space Operators

```python
element_order = ['Q0', 'DT1', 'Q1']
dims = [6, 12, 6]  # Total: 432

# Bare Hamiltonian
for element in element_order:
    H_list = [H if key == element else I for key in element_order]
    Hs.append(kron_it(H_list))
H_bare = sum(Hs)

# Number operators (for state identification)
for element in element_order:
    N_list = [N if key == element else I for key in element_order]
    Ns.append(kron_it(N_list))
```

### Step 3: Build Coupling Hamiltonian H_c

```python
H_c = np.zeros((432, 432))

for coupling in [['Q0', 'DT1;0', 0.18], ['Q1', 'DT1;1', 0.18], ...]:
    Q1_var, Q2_var, g = coupling

    # Get impedances for coupling coefficient
    Z1 = op_dict[Q1_var.split(';')[0]]['Z']
    Z2 = op_dict[Q2_var.split(';')[0]]['Z']
    coupling_coef = 8 * e**2 * g * sqrt(Z1 * Z2) / hbar

    # Build Q1 x Q2 in full space
    Q_list = []
    for key in element_order:
        if key == Q1_var.split(';')[0]:
            Q_list.append(Q_operator_for_Q1_var)
        elif key == Q2_var.split(';')[0]:
            Q_list.append(Q_operator_for_Q2_var)
        else:
            Q_list.append(I)

    H_c += coupling_coef * kron_it(Q_list)
```

**Key insight:** H_c contains ONLY the coupling terms, not the bare Hamiltonians.

### Step 4: Build Drive Charge Operators

```python
Q_drive_full = {}

for element in ['Q0', 'Q1']:
    Q_op = op_dict[element]['Q']
    Q_list = [Q_op if key == element else I for key in element_order]
    Q_drive_full[element] = kron_it(Q_list)
```

This gives:
- `Q_Q0_full = Q_Q0 x I_DTC x I_Q1` (432 × 432)
- `Q_Q1_full = I_Q0 x I_DTC x Q_Q1` (432 × 432)

### Step 5: Diagonalize Total Hamiltonian

```python
H_total = H_bare + H_c
eigenvalues, V = np.linalg.eigh(H_total)
eigenvalues -= eigenvalues[0]  # Reference to ground state
```

V is 432 × 432:
- Column k = eigenvector |ψ_k⟩ in bare basis
- Sorted by energy (column 0 = ground state)

### Step 6: Transform Operators to Dressed Basis

```python
# Coupling matrix: H_c in dressed basis
M_coupling_full = V.conj().T @ H_c @ V
M_coupling = np.abs(M_coupling_full[:N_LEVELS, :N_LEVELS]) * 1000  # GHz -> MHz

# Transition matrices: Q_qubit in dressed basis
for elem in ['Q0', 'Q1']:
    M_elem = V.conj().T @ Q_drive_full[elem] @ V
    transition_matrices[elem] = np.abs(M_elem[:N_LEVELS, :N_LEVELS])
```

---

## Physical Interpretation

### Coupling Matrix |⟨ψ_i|H_c|ψ_j⟩|

| Element | Meaning |
|---------|---------|
| **M[i,i]** | Energy shift of state i due to coupling (often absorbed into eigenvalue) |
| **M[i,j] large** | States i and j are strongly hybridized by H_c |
| **M[i,j] ~ 0** | States i and j don't interact via coupling |

**Use case:** Understanding avoided crossings and state mixing.

### Transition Matrix |⟨ψ_i|Q_qubit|ψ_j⟩|

| Element | Meaning |
|---------|---------|
| **M[i,j] large** | Can drive i<->j transition via qubit XY line |
| **M[i,j] ~ 0** | Transition is dark to qubit drive |

**Use case:** Predicting which transitions are accessible via direct qubit drive (not through resonator).

---

## Comparison: This Tool vs Spectroscopy Prediction

### System Differences

| Aspect | Coupling Matrix Viz | Spectroscopy Prediction |
|--------|--------------------|-----------------------|
| **System** | Q-DTC-Q | R-Q-DTC-Q-R |
| **Hilbert dim** | 6 × 12 × 6 = 432 | 3 × 6 × 12 × 6 × 3 = 3888 |
| **Has resonators?** | No | Yes |

### Operator Differences

| Tool | Operator | Formula | Physical Meaning |
|------|----------|---------|------------------|
| **Coupling Matrix** | H_c | Sum of g_ij Q_i x Q_j | Static inter-element coupling |
| **Transition Matrix** | Q_qubit | Q_Q0 x I x I | Direct qubit drive |
| **Spectroscopy** | Q_resonator | I x I x I x (a+a†) x I | Resonator drive |

### Output Differences

| Tool | Matrix Elements Used | Output Format |
|------|---------------------|---------------|
| **Coupling Matrix** | ALL M[i,j] | N×N heatmap |
| **Transition Matrix** | ALL M[i,j] | N×N heatmap |
| **Spectroscopy** | Only M[0,k] (ground state row) | 2D scatter (phi vs freq) |

### When to Use Each

| Question | Tool |
|----------|------|
| "What will I see in resonator spectroscopy?" | **Spectroscopy Prediction** |
| "Why do these states have an avoided crossing?" | **Coupling Matrix** |
| "Can I drive this transition with a qubit pi-pulse?" | **Transition Matrix** |
| "Which states are dark to resonator but bright to qubit?" | Compare **Spectroscopy** vs **Transition** |

---

## Key Code Differences

### Spectroscopy Prediction (dtc_spectroscopy_prediction)

```python
# Uses resonator (a+a†) operator
Q_bare = result['resonator_Q_ops']['R0']  # 3888 × 3888

# Only extracts ground state transitions
M = V.conj().T @ Q_bare @ V
strengths = np.abs(M[0, :])**2   # Row 0 only
freqs = E - E[0]

# Output: (phi, freq, strength) tuples for plotting
```

### Coupling/Transition Matrix (this tool)

```python
# Uses H_c or Q_qubit operator
H_c = build_coupling_hamiltonian()  # 432 × 432
Q_Q0_full = kron_it([Q_Q0, I, I])   # 432 × 432

# Extracts FULL matrix
M_coupling = V.conj().T @ H_c @ V
M_transition = V.conj().T @ Q_Q0_full @ V

# Output: N×N matrix at each flux point
```

---

## Visualization Features

### Interactive Elements

1. **Flux Slider**: Sweep phi from 0 to 0.5 Phi_0
2. **Hover Info Panel**: Shows for any (i,j) element:
   - State compositions of i and j
   - <N> expectation values
   - Energy difference Delta_E
   - Matrix element value
3. **Energy Diagram**: Right panel shows energy levels vs flux with cursor
4. **Color Coding**: States colored by dominant character (Q0=blue, DTC=orange, Q1=green)

### Diagonal Elements

- **Coupling Matrix**: Shows energy (GHz) as text annotation
- **Transition Matrix**: Shows energy (GHz) as text annotation
- Off-diagonal color scale excludes diagonal (set to NaN)

---

## File Structure

```
coupling_matrix_viz/
├── coupling_matrix_interactive.py   # Main script (all modes)
├── outputs/
│   ├── coupling_matrix_interactive.html
│   ├── transition_matrix_Q0_interactive.html
│   └── transition_matrix_Q1_interactive.html
├── README.md
└── TECHNICAL_DETAILS.md
```

---

## Parameters

```python
N_LEVELS = 20        # Number of lowest energy levels to display
N_FLUX_POINTS = 51   # Flux points (0 to 0.5)
N_CORES = 28         # Parallel cores for sweep

simDict = {
    'transmons': {
        'Q0':  {'f01': 4.5, 'anharm': 0.21, 'cutoff': 6},
        'DT1': {'f01': 3.8, 'anharm': 0.08, 'g': 0.7, 'phi': phi, 'cutoff': 12},
        'Q1':  {'f01': 5.0, 'anharm': 0.21, 'cutoff': 6},
    },
    'couplings': [
        ['Q0', 'DT1;0', 0.18],
        ['Q1', 'DT1;1', 0.18],
        ['DT1;0', 'DT1;1', 0.01],
    ],
    'N': 12,
}
```

---

## Key Formulas

| Quantity | Formula | Code |
|----------|---------|------|
| Coupling matrix | M_c = V† H_c V | `V.conj().T @ H_c @ V` |
| Transition matrix | M_t = V† Q V | `V.conj().T @ Q_full @ V` |
| Coupling coefficient | g_eff = 8e²g√(Z₁Z₂)/ℏ | `8 * e**2 * g * np.sqrt(Z1*Z2) / hbar` |
| State character | argmax(<N_i>) | `np.argmax(exp_values)` |
