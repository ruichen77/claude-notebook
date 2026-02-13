# Quantum Simulation Concepts Reference

A quick reference guide for key concepts in superconducting circuit simulation.

> **See also:** [[DTC State Labeling]], [[Rockbottom Gotchas]], [[Spectroscopy Prediction Workflow]], [[Coupling and Transition Matrix Visualization]], [[Dispersive Coupler Implementation Plan]]

---

## 1. Core Quantum Mechanics Concepts

### Operators, States, and Eigenvalues

| Concept | What it is | Type | Example |
|---------|------------|------|---------|
| **Operator** | Represents a measurable quantity | Matrix | H (432×432) |
| **State** | Represents the system configuration | Vector | \|ψ⟩ (432×1) |
| **Eigenvalue** | The measurement result | Scalar | E = 4.5 GHz |

**The eigenvalue equation:**
```
H |ψ_k⟩ = E_k |ψ_k⟩

Operator × Eigenstate = Eigenvalue × Eigenstate
```

### Eigenstates / Eigenvectors / Eigenenergies

These describe the **solutions** to the eigenvalue equation:

| Term | What it is | In code |
|------|------------|---------|
| **Eigenstate \|ψ_k⟩** | A quantum state with definite energy | Conceptual |
| **Eigenvector** | Mathematical representation of eigenstate | `V[:, k]` |
| **Eigenenergy E_k** | The energy of that state | `eigenvalues[k]` |

**Key property:** Eigenstates are **stable** — they don't evolve into other states over time.

### Observables

An **observable** is any physical quantity you can measure. Each is represented by an operator.

| Observable | Operator | What you measure |
|------------|----------|------------------|
| Energy | H (Hamiltonian) | System's total energy |
| Position/Field | x̂ or (a + a†) | Field amplitude |
| Number | n̂ | Photon or Cooper pair count |
| Charge | Q | Charge on a capacitor |

**Important:** H is just ONE observable. Many measurements are NOT H!

---

## 2. The Hamiltonian

### What is the Hamiltonian?

The **Hamiltonian H** is the **total energy operator** of the system.

```
H = H_kinetic + H_potential
```

| Concept | What it represents |
|---------|-------------------|
| **Hamiltonian H** | Total energy operator |
| **Eigenvalues of H** | Allowed energy levels |
| **Eigenstates of H** | Stable states with definite energy |

### Why H is Special

1. **Time evolution:** H generates time evolution via U(t) = exp(-iHt/ℏ)
2. **Stability:** Eigenstates of H don't change over time (only phase rotates)
3. **Energy conservation:** Energy is conserved, so H eigenstates are stable

### Transmon Hamiltonian

```python
H_transmon = H_capacitive + H_Josephson
```

| Term | Formula | Physical meaning |
|------|---------|------------------|
| H_capacitive | 4Ec(n - ng)² | Energy stored in capacitor (diagonal) |
| H_Josephson | -Ej/2 (|n⟩⟨n+1| + h.c.) | Cooper pair tunneling (off-diagonal) |

**H_transmon is in CHARGE basis** — must diagonalize to get energy eigenstates.

---

## 3. Bases and Transformations

### Two Key Bases

| Basis | States | When H is diagonal? |
|-------|--------|---------------------|
| **Charge basis** | \|n⟩ = Cooper pair number | No (must diagonalize) |
| **Energy basis** | \|0⟩, \|1⟩, \|2⟩... = eigenstates | Yes (by definition) |

### The Diagonalization Process

```
CHARGE BASIS                          ENERGY BASIS
(25 states)                           (6 states after truncation)

|n=-12⟩                               |0⟩  (ground)
|n=-11⟩                               |1⟩  (first excited)
  ⋮           ──── diagonalize ────►  |2⟩
|n=0⟩                                 |3⟩
  ⋮                                   |4⟩
|n=+12⟩                               |5⟩

H_transmon                            H_diagonal
(tridiagonal, 25×25)                  (diagonal, 6×6)
```

```python
# Diagonalization
eigenvalues, V = np.linalg.eig(H_transmon)

# V is the transformation matrix:
# - Columns of V are eigenvectors
# - V[:, k] = |ψ_k⟩ expressed in charge basis
```

### Bare vs Dressed Basis (Full System)

| Basis | Description | States |
|-------|-------------|--------|
| **Bare product basis** | Tensor product of individual energy bases | \|n_Q0, n_DTC, n_Q1⟩ |
| **Dressed basis** | Eigenstates of full coupled H | \|ψ_0⟩, \|ψ_1⟩, \|ψ_2⟩... |

Coupling mixes bare states → dressed states are superpositions:
```
|ψ_1⟩ = 0.85|0,1,0⟩ + 0.12|0,2,0⟩ + 0.03|1,0,0⟩ + ...
```

---

## 4. Key Matrices in Simulation

### Summary Table

| Matrix | Dimensions | Basis | Description |
|--------|------------|-------|-------------|
| `H_transmon` | 25×25 | Charge | Single transmon Hamiltonian |
| `H_total` | 432×432 or 3888×3888 | Bare product | Full system Hamiltonian |
| `V` | N×N | Columns=dressed, Rows=bare | Eigenvector matrix |
| `Q_bare` | N×N | Bare product | Operator before transformation |
| `M_dressed` | N×N | Dressed | Operator after V†QV transformation |

### The Eigenvector Matrix V

**V is a matrix, NOT a vector.** Each column is one eigenvector.

```
V = N × N matrix

         │  |ψ_0⟩   |ψ_1⟩   |ψ_2⟩  ...  │
         │   ↓       ↓       ↓          │
         ┌─────────────────────────────────┐
|bare_0⟩ │  v_00    v_01    v_02   ...   │  ← row 0
|bare_1⟩ │  v_10    v_11    v_12   ...   │  ← row 1
|bare_2⟩ │  v_20    v_21    v_22   ...   │  ← row 2
   ⋮     │   ⋮       ⋮       ⋮      ⋱    │
         └─────────────────────────────────┘
              ↑
           Column k = eigenvector |ψ_k⟩
                      expressed in bare basis
```

**Index meanings:**
- **Column k**: The k-th dressed eigenstate (sorted by energy)
- **Row i**: Coefficient of bare basis state |bare_i⟩
- **V[i, k]**: ⟨bare_i|ψ_k⟩ = amplitude of bare state i in dressed state k

---

## 5. Operators: Q_qubit vs Q_resonator

### Comparison

| Property | **Q_qubit** (charge) | **Q_resonator** (field) |
|----------|---------------------|-------------------------|
| Physical meaning | Cooper pair number n̂ | Position quadrature (a + a†) |
| Original basis | Charge basis | Fock basis |
| Construction | Diagonalize H, transform n̂ | Direct: a + a† |
| Matrix structure | Dense | Tri-diagonal |
| Selection rules | No simple rule | Δn = ±1 only |

### Q_qubit Construction (requires diagonalization)

```python
# Step 1: Charge operator in CHARGE basis
Q_charge = np.diag([-12, -11, ..., 0, ..., 11, 12])  # 25×25, diagonal

# Step 2: Diagonalize transmon Hamiltonian
eigenvalues, V = np.linalg.eig(H_transmon)

# Step 3: Transform Q to ENERGY eigenbasis
Q_energy = V.conj().T @ Q_charge @ V  # Now dense!

# Step 4: Truncate
Q_qubit = Q_energy[:6, :6]  # 6×6
```

### Q_resonator Construction (no diagonalization needed)

```python
# Direct in Fock basis (already energy eigenbasis for harmonic oscillator)
a = np.diag(np.sqrt([1, 2]), k=1)  # Lowering operator
Q_resonator = a + a.T

# Q_resonator = [[0, 1,   0  ],
#                [1, 0,   √2 ],
#                [0, √2,  0  ]]
```

**Why different?** Resonator is harmonic → Fock basis = energy basis. Transmon is anharmonic → must diagonalize first.

---

## 6. Matrix Element Transformation

### The Formula: M = V† Q V

To compute matrix elements of operator Q between dressed eigenstates:

```python
M_dressed = V.conj().T @ Q_bare @ V
# M_dressed[i, j] = ⟨ψ_i|Q|ψ_j⟩
```

### What M_dressed Represents

```
M_dressed[i,j] = ⟨ψ_i| Q |ψ_j⟩

         |ψ_0⟩   |ψ_1⟩   |ψ_2⟩   |ψ_3⟩  ...
        ┌──────┬──────┬──────┬──────┬─────
|ψ_0⟩   │  ~0  │ M_01 │ M_02 │ M_03 │ ...
        ├──────┼──────┼──────┼──────┼─────
|ψ_1⟩   │ M_10 │  ~0  │ M_12 │ M_13 │ ...
        ├──────┼──────┼──────┼──────┼─────
  ⋮     │  ⋮   │  ⋮   │  ⋮   │  ⋮   │ ⋱
```

**Important:** M_dressed is NOT the Hamiltonian!

| Matrix | Diagonal | Off-diagonal |
|--------|----------|--------------|
| **H_dressed** | Eigenvalues E_k | Zero (by definition) |
| **M_dressed = V†QV** | Usually ~0 | Transition amplitudes |

---

## 7. Spectroscopy Prediction Workflow

### Pipeline Overview

```
FOR EACH FLUX POINT φ:

1. simDict + φ         →  Device parameters
2. simOne()            →  Diagonalize H_total (3888×3888)
                          Get eigenvalues E_k, eigenvectors V
3. Build Q_bare        →  Q_R0 = (a+a†) ⊗ I ⊗ I ⊗ I ⊗ I
4. M = V† Q V          →  Transform to dressed basis
5. Extract |M|²        →  strengths[k] = |M[0,k]|²
                          freqs[k] = E_k - E_0
6. Plot 2D map         →  x = φ, y = freq, color = |M|²
```

### Key Quantities

| Quantity | Formula | Meaning |
|----------|---------|---------|
| Transition frequency | f_k = E_k - E_0 | Energy to excite state k |
| Transition strength | \|M[0,k]\|² = \|⟨ψ_k\|Q\|ψ_0⟩\|² | How bright in spectroscopy |
| Sum rule | Σ_k \|M[0,k]\|² ≈ 1 | Completeness check |

---

## 8. Three Different Matrix Visualizations

### Comparison Table

| Tool | Operator | System | Shows |
|------|----------|--------|-------|
| **Spectroscopy** | Q_R0 = (a+a†) | R-Q-DTC-Q-R | Drive via resonator |
| **Coupling Matrix** | H_c | Q-DTC-Q | State hybridization |
| **Transition Matrix** | Q_qubit | Q-DTC-Q | Drive via qubit line |

### When to Use Each

| Question | Tool |
|----------|------|
| "What will I see in resonator spectroscopy?" | **Spectroscopy Prediction** |
| "Why do these states have an avoided crossing?" | **Coupling Matrix** |
| "Can I drive this transition via qubit XY line?" | **Transition Matrix** |

---

## 9. Quick Reference: Dimensions

### Q-DTC-Q System (no resonators)

```
Q0:  6 levels (cutoff)
DTC: 12 levels (cutoff)
Q1:  6 levels (cutoff)

Total: 6 × 12 × 6 = 432 dimensions
```

### R-Q-DTC-Q-R System (with resonators)

```
R0:  3 levels
Q0:  6 levels
DTC: 12 levels
Q1:  6 levels
R1:  3 levels

Total: 3 × 6 × 12 × 6 × 3 = 3,888 dimensions
```

---

## 10. Common Gotchas

### 1. State Labels Show Dominant Component Only

```python
# Label "01000" only shows the DOMINANT bare state
# Actual state may be:
|ψ⟩ = 0.85|0,1,0,0,0⟩ + 0.12|0,2,0,0,0⟩ + ...

# Use eigenvector V[:, k] for true composition!
```

### 2. V is a Matrix, Not a Vector

```python
V.shape = (3888, 3888)  # Matrix!
V[:, k]  # Column k = eigenvector for state k (this is a vector)
```

### 3. M_dressed is NOT the Hamiltonian

```python
M = V† Q V  # This is Q in dressed basis, NOT energies!
# Diagonal of M ≠ eigenvalues
# Diagonal of M ≈ 0 for (a+a†) operator
```

### 4. Q_qubit vs Q_resonator are Different

```python
Q_resonator = a + a†     # Tri-diagonal, Δn = ±1 selection rule
Q_qubit = V† n̂ V         # Dense, no simple selection rule
```

---

## 11. Key Formulas

| Quantity | Formula | Code |
|----------|---------|------|
| Diagonalization | H\|ψ_k⟩ = E_k\|ψ_k⟩ | `eigenvalues, V = np.linalg.eig(H)` |
| Basis transformation | M = V†QV | `M = V.conj().T @ Q @ V` |
| Transition frequency | f_k = E_k - E_0 | `freqs = E - E[0]` |
| Transition strength | \|⟨ψ_k\|Q\|ψ_0⟩\|² | `strengths = np.abs(M[0, :])**2` |
| Kronecker product | A ⊗ B | `np.kron(A, B)` |
| Time evolution | U(t) = e^(-iHt/ℏ) | `scipy.linalg.expm(-1j * H * t)` |

---

## 12. File Locations

| Repository | Purpose | Location |
|------------|---------|----------|
| EnhancedSmarterThanARock | Core simulation library | `landsman2:~/repos/EnhancedSmarterThanARock/` |
| dtc_spectroscopy_prediction | [[Spectroscopy Prediction Workflow]] | `landsman2:~/repos/dtc_spectroscopy_prediction/` |
| coupling_matrix_viz | [[Coupling and Transition Matrix Visualization]] | `landsman2:~/repos/coupling_matrix_viz/` |
| dispersive_shift_calculator | χ calculations — see [[Parallel Simulation Benchmarks]] | `landsman2:~/repos/dispersive_shift_calculator/` |
