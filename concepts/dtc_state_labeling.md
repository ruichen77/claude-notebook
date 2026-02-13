# DTC Simulation Notes: Understanding State Labeling

> **See also:** [[Quantum Simulation Concepts]], [[Rockbottom Gotchas]], [[Dispersive Coupler Implementation Plan]]

## Summary

When using the `rockbottom` simulator for Q-DTC-Q systems, there is an important distinction between **physical photon counting** and **energy-ordered state indexing** for the Double Transmon Coupler (DTC).

---

## The Two Definitions of "Excitation Number"

### 1. Physical Photon Count (Bare Junction Basis)

Count excitations in the **bare junction basis** |n₀, n₁⟩ **before** DTC internal diagonalization:

| Manifold | States | Count |
|----------|--------|-------|
| 0-photon | \|0,0⟩ | 1 |
| 1-photon | \|1,0⟩, \|0,1⟩ | 2 |
| 2-photon | \|2,0⟩, \|1,1⟩, \|0,2⟩ | 3 |
| 3-photon | \|3,0⟩, \|2,1⟩, \|1,2⟩, \|0,3⟩ | 4 |

This reflects the physical degrees of freedom of the two junctions.

### 2. Energy-Ordered State Index (What `rockbottom` Returns)

The `get_single_DT` function in `rockbottom.py`:
1. Builds the coupled DTC Hamiltonian in the charge basis
2. **Pre-diagonalizes** the DTC subsystem
3. **Truncates** to keep only `cutoff` lowest eigenstates
4. Returns dressed eigenstates |0⟩, |1⟩, |2⟩, |3⟩...

The returned number operator is simply:
```python
N = np.diag([0, 1, 2, 3, ...])  # Just the energy level index!
```

**This is NOT the physical photon number** - it's the energy-ordered eigenstate index of the pre-diagonalized DTC.

---

## Why This Matters

When you see a state labeled |0,2,0⟩ in the simulation output (Q0=0, DTC=2, Q1=0):
- The "2" means **the 3rd energy eigenstate of the pre-diagonalized DTC**
- It does NOT mean "2 total photons in the DTC junctions"
- This dressed state is a superposition of many bare |n₀, n₁⟩ states

### Consequence for 2-Photon Manifold Analysis

With strong internal DTC coupling (e.g., g = 0.7 GHz >> anharmonicity 0.08 GHz):
- The three bare 2-photon states |2,0⟩, |1,1⟩, |0,2⟩ hybridize strongly
- After diagonalization, they become dressed states spread across multiple energy levels
- Searching for states with ⟨N_DTC⟩ ≈ 2 finds only the **energy eigenstate index**, not physical 2-photon states

---

## How to Properly Count Physical Photons

To identify states by their **physical photon content** (total bare junction excitations), you need to:

### Option 1: Modify `get_single_DT` (Recommended)
Return the bare junction number operators N₀ and N₁ transformed into the truncated eigenbasis:
```python
# In get_single_DT, before returning:
N0_bare = kron_it([N_single, Qeye])  # Junction 0 number op
N1_bare = kron_it([Qeye, N_single])  # Junction 1 number op

# Transform to eigenbasis
N0_eig = v.conj().T @ N0_bare @ v
N1_eig = v.conj().T @ N1_bare @ v

# Return truncated versions
return He, Q_list, I, N, N0_eig[:cutoff,:cutoff], N1_eig[:cutoff,:cutoff]
```

Then in the full system, you can compute ⟨N₀⟩ + ⟨N₁⟩ to get true photon count.

### Option 2: Build Without Pre-diagonalization
Construct the full Q0 ⊗ (bare DTC) ⊗ Q1 system in the charge basis, then diagonalize everything at once. This preserves direct access to junction occupation numbers.

### Option 3: Track the Transformation Matrix
Keep the `v` matrix from DTC diagonalization to map between eigenbasis and bare basis.

---

## Code Location

- `rockbottom.py`: `/home/US8J4928/repos/EnhancedSmarterThanARock/enhancedSmarterThanARock/rockbottom.py`
- Key function: `get_single_DT(Ej, Ec, ng, g, phi, Z, ...)`
- The pre-diagonalization happens at:
  ```python
  (w, v) = np.linalg.eig(H)
  ind = np.argsort(np.real(w))
  ```
- The "number operator" is just:
  ```python
  return He, [...], np.eye(maxl), np.diag(list(range(0, maxl)))
  ```

---

## Related Files

- Dispersive shift calculator: `/home/US8J4928/repos/dispersive_shift_calculator/`
- Example analysis script: `example_chi_analysis.py`

---

## Naming Convention Update (Jan 2026)

The dispersive_shift_calculator library has been updated to use correct terminology:
- **Old (misleading)**: "n-photon", "photon count", "excitation"
- **New (correct)**: "level n", "energy index", "energy-ordered eigenstate index"

All code and comments now clarify that state labels use **energy-ordered indices**, not physical photon counts.

---

## TODO

- [ ] Implement Option 1 to get true junction number operators
- [ ] Verify by checking that 2-photon manifold has exactly 3 states
- [ ] Update dispersive shift calculation to use physical photon counting
