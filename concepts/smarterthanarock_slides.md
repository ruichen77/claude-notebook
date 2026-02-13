# SmarterThanARock Simulator: Hamiltonian Construction Flow

## 5 Slides for Presentation

---

# Slide 1: Overview - From Circuit to Energy Levels

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SIMULATION PIPELINE OVERVIEW                         │
└─────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │   Circuit    │      │  Subsystem   │      │    Full      │
    │  Parameters  │ ──▶  │ Hamiltonians │ ──▶  │ Hamiltonian  │
    │  (simDict)   │      │  (rockbottom)│      │  (Kronecker) │
    └──────────────┘      └──────────────┘      └──────────────┘
                                                       │
                                                       ▼
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │   Output:    │      │    State     │      │   Numerical  │
    │ Energy Dict  │ ◀──  │   Labeling   │ ◀──  │Diagonalization
    │   (edict)    │      │              │      │  (np.linalg) │
    └──────────────┘      └──────────────┘      └──────────────┘
```

**Key Input: simDict**
```python
simDict = {
    'transmons': {
        'Q0':  {'f01': 4.5, 'anharm': 0.21, 'cutoff': 6},
        'DT1': {'f01': 3.8, 'anharm': 0.08, 'g': 0.7, 'phi': 0.4, 'cutoff': 12},
        'Q1':  {'f01': 5.0, 'anharm': 0.21, 'cutoff': 6}
    },
    'resonators': {
        'R0': {'f_r': 7.04, 'n_levels': 3},
        'R1': {'f_r': 7.2,  'n_levels': 3}
    },
    'couplings': [['Q0', 'DT1;0', 0.18], ['Q1', 'DT1;1', 0.18], ...],
    'N': 12  # Charge basis size
}
```

**Key Output: edict** (energy dictionary)
```python
edict = {'00000': 0.0, '10000': 4.5, '01000': 3.8, '00100': 5.0, ...}
#        Labels: (R0, Q0, DTC, Q1, R1) → Energy (GHz)
```

---

# Slide 2: Single Transmon Hamiltonian Construction

```
┌─────────────────────────────────────────────────────────────────────────┐
│              TRANSMON: CHARGE BASIS → ENERGY EIGENBASIS                 │
└─────────────────────────────────────────────────────────────────────────┘
```

**Step 1: Build in Charge Basis** (dimension = 2N+1, typically N=12 → 25 states)

$$H_{charge} = 4E_C \sum_n (n - n_g)^2 |n\rangle\langle n| - \frac{E_J}{2}\sum_n (|n\rangle\langle n+1| + |n+1\rangle\langle n|)$$

Where:
- $E_C$ = charging energy ≈ anharmonicity (e.g., 0.21 GHz for qubits)
- $E_J$ = Josephson energy, derived from: $E_J = \frac{(f_{01} + E_C)^2}{8 E_C}$
- $n_g$ = gate charge offset (typically 0)
- Charge states: $n \in [-N, N]$ → 25 basis states

**Step 2: Diagonalize & Truncate** (25 states → cutoff states)

```python
(eigenvalues, eigenvectors) = np.linalg.eig(H_charge)
# Sort by energy, keep lowest 'cutoff' states
# Transform charge operator Q to energy eigenbasis
```

**Typical Truncation Levels:**
| Element | Charge Basis (2N+1) | Energy Cutoff | Physical Reason |
|---------|---------------------|---------------|-----------------|
| Qubit   | 25 states          | 6 levels      | Only need |0⟩-|5⟩ |
| DTC     | 25 states          | 12 levels     | Need higher states for 2-excitation manifold |
| Resonator | N/A (direct)     | 3 levels      | Linear oscillator, low photon number |

**Output Operators** (all in truncated energy eigenbasis):
- $H_e$ = diagonal energy matrix (ground state shifted to 0)
- $Q$ = charge operator (for coupling to other elements)
- $I$ = identity matrix
- $N$ = number operator = diag(0, 1, 2, ...)

---

# Slide 3: Double-Transmon Coupler (DTC) Hamiltonian

```
┌─────────────────────────────────────────────────────────────────────────┐
│              DTC: TWO COUPLED TRANSMONS WITH FLUX TUNING                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Physical Picture:**
```
      ┌─────────┐         ┌─────────┐
      │Transmon │───g─────│Transmon │
      │   #1    │   ↑     │   #2    │
      └─────────┘   │     └─────────┘
                    │
              Flux Φ/Φ₀ = φ
              (tunes coupling)
```

**Hamiltonian Construction:**

$$H_{DTC} = H_1 \otimes I_2 + I_1 \otimes H_2 + H_{coupling}$$

**Coupling Term** (flux-dependent):

$$H_{coupling} = -g_{eff} \left[ \cos(2\pi\phi)(X_1 X_2 + Y_1 Y_2) + \sin(2\pi\phi)(Y_1 X_2 - X_1 Y_2) \right]$$

Where:
- $g$ = internal DTC coupling strength (e.g., 0.7 GHz)
- $\phi$ = external flux in units of $\Phi_0$ (tunable: 0 → 0.5)
- $X, Y$ = Pauli-like operators from charge basis Fourier components

**Hilbert Space:**
- Before truncation: 25 × 25 = 625 states
- After truncation: cutoff² (e.g., 12 × 12 = 144 → truncated to 12 states)

**Output Operators:**
- $H_{DTC}$ = diagonalized, truncated Hamiltonian
- $Q_1$ = charge operator for transmon #1 (for coupling to Q0)
- $Q_2$ = charge operator for transmon #2 (for coupling to Q1)
- $Q_1 Q_2$ = product operator (for internal correlations)

**Key Parameters:**
```python
'DT1': {
    'f01': 3.8,      # Base frequency (GHz)
    'anharm': 0.08,  # Anharmonicity (GHz) - smaller than qubits!
    'g': 0.7,        # Internal coupling (GHz)
    'phi': 0.4,      # Flux point (Φ/Φ₀)
    'cutoff': 12     # Truncation level
}
```

---

# Slide 4: Full Hilbert Space Assembly via Kronecker Products

```
┌─────────────────────────────────────────────────────────────────────────┐
│              KRONECKER PRODUCT: COMBINING ALL SUBSYSTEMS                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Element Order** (determines state label indexing):
```
element_order = ['R0', 'Q0', 'DT1', 'Q1', 'R1']
                  ↓     ↓     ↓     ↓     ↓
dimensions:       3  ×  6  × 12  ×  6  ×  3  = 3,888 total states
```

**Bare Hamiltonian** (uncoupled sum):

$$H_{bare} = H_{R0} \otimes I \otimes I \otimes I \otimes I + I \otimes H_{Q0} \otimes I \otimes I \otimes I + \cdots$$

In code:
```python
def kron_it(op_list):
    """Recursively compute tensor product of list of operators"""
    result = op_list[0]
    for op in op_list[1:]:
        result = np.kron(result, op)
    return result

# Example: Q0's Hamiltonian in full space
H_Q0_full = kron_it([I_R0, H_Q0, I_DT1, I_Q1, I_R1])
```

**Coupling Terms:**

For coupling `['Q0', 'DT1;0', g=0.18]`:

$$H_{coupling} = g_{eff} \cdot (I_{R0} \otimes Q_{Q0} \otimes I_{DT1} \otimes I_{Q1} \otimes I_{R1}) \cdot (I_{R0} \otimes I_{Q0} \otimes Q_{DT1,0} \otimes I_{Q1} \otimes I_{R1})$$

Where $g_{eff} = \frac{8e^2 g}{\hbar} \sqrt{Z_1 Z_2}$ includes impedance matching.

**Total Hamiltonian:**

$$H_{total} = H_{bare} + \sum_{couplings} H_{coupling}$$

**Hilbert Space Dimensions by System:**
| System | Dimensions | Matrix Size |
|--------|------------|-------------|
| Q-DTC-Q | 6 × 12 × 6 | 432 × 432 |
| R-Q-DTC-Q-R | 3 × 6 × 12 × 6 × 3 | 3,888 × 3,888 |

---

# Slide 5: Diagonalization & State Labeling

```
┌─────────────────────────────────────────────────────────────────────────┐
│              EIGENDECOMPOSITION & OUTPUT GENERATION                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Step 1: Numerical Diagonalization**

```python
eigenvalues, eigenvectors = np.linalg.eig(H_total)  # 3888 × 3888 matrix
# Sort by increasing energy
idx = np.argsort(np.real(eigenvalues))
eigenvalues = eigenvalues[idx]
eigenvectors = eigenvectors[:, idx]
# Shift ground state to zero
eigenvalues -= eigenvalues[0]
```

**Step 2: State Labeling via Dominant Basis State**

For each eigenstate |ψ_k⟩, find which bare basis state has the largest coefficient:

```python
for k in range(n_states):
    psi = eigenvectors[:, k]           # Eigenvector
    max_idx = np.argmax(np.abs(psi)**2)  # Dominant basis index

    # Convert flat index to occupation numbers using number operators
    # N_R0, N_Q0, N_DTC, N_Q1, N_R1 evaluated at max_idx
    label = f"{n_R0}{n_Q0}{n_DTC}{n_Q1}{n_R1}"  # e.g., "01200"
```

**⚠️ CRITICAL GOTCHA:**
The label shows the **dominant bare state only**, NOT the full wavefunction!

Example of a hybridized state:
```
|ψ⟩ = 0.85|0,1,0,0,0⟩ + 0.32|0,2,0,0,0⟩ + 0.15|0,1,1,0,0⟩
Label: "01000" (only shows dominant component!)
```

For true composition, must analyze eigenvector directly.

**Step 3: Output Dictionary**

```python
edict = {
    '00000': 0.000,   # Ground state
    '01000': 4.523,   # Q0 excited
    '00100': 3.812,   # DTC |1⟩
    '00010': 5.034,   # Q1 excited
    '00200': 7.651,   # DTC |2⟩
    '01100': 8.312,   # Q0 + DTC
    '10000': 7.040,   # R0 |1⟩
    ...
}
```

**Transition Matrix Elements** (for spectroscopy):

$$M_{0 \to k} = \langle \psi_k | Q_{R0} + e^{i\delta} Q_{R1} | \psi_0 \rangle$$

- Single drive: Use $Q_{R0}$ or $Q_{R1}$ alone
- Dual drive: Phase $\delta$ controls which transitions are enhanced
  - $\delta = 0$: Symmetric → N-like DTC modes
  - $\delta = \pi$: Antisymmetric → T-like DTC modes

---

# Summary: Key Numbers to Remember

| Parameter | Typical Value | Physical Meaning |
|-----------|--------------|------------------|
| Qubit f₀₁ | 4-5 GHz | Qubit transition frequency |
| Qubit anharm | 0.21 GHz | Distinguishes |1⟩→|2⟩ from |0⟩→|1⟩ |
| DTC anharm | 0.08 GHz | Smaller → more harmonic |
| DTC internal g | 0.7 GHz | Strong internal coupling |
| Q-DTC coupling | 0.18 GHz | Qubit-coupler interaction |
| Charge basis N | 12 | 25 charge states per transmon |
| Qubit cutoff | 6 | Keep 6 lowest energy levels |
| DTC cutoff | 12 | Need more levels for 2-excitation physics |
| Resonator levels | 3 | |0⟩, |1⟩, |2⟩ photon states |

**Total Hilbert Space: 3 × 6 × 12 × 6 × 3 = 3,888 dimensions**

---
