# Dispersive Shift Calculation for DTC Coupler States

## Implementation Plan for `dispersive_coupler.py`

**Purpose:** Calculate qubit frequency shift (χ) due to DTC coupler excitation to higher photon manifolds (states 3, 4, 5, etc.)

**Package:** EnhancedSmarterThanARock

---

## 1. Physics Background

### Core Formula

Dispersive shift for DTC eigenstate k:

```
χ^(k) = E_|1,k,0⟩ - E_|0,k,0⟩ - E_|1,0,0⟩ + E_|0,0,0⟩
```

Four states required:
| State | Label | Meaning |
|-------|-------|---------|
| E_|0,0,0⟩ | "000" | Qubit g, DTC ground |
| E_|1,0,0⟩ | "100" | Qubit e, DTC ground |
| E_|0,k,0⟩ | "0k0" | Qubit g, DTC in eigenstate k |
| E_|1,k,0⟩ | "1k0" | Qubit e, DTC in eigenstate k |

### DTC Excitation Manifolds

After DTC pre-diagonalization in `get_single_DT()`:

| DTC Index | Excitation Manifold | Bare State Composition |
|-----------|--------------------|-----------------------|
| 0 | Ground | \|00⟩ |
| 1, 2 | 1-excitation | Mix of \|10⟩, \|01⟩ |
| 3, 4, 5 | 2-excitation | Mix of \|20⟩, \|11⟩, \|02⟩ |
| 6, 7, 8, 9 | 3-excitation | Mix of \|30⟩, \|21⟩, \|12⟩, \|03⟩ |

**Key limitation:** Internal composition (which mix of \|20⟩, \|11⟩, \|02⟩) is lost after pre-diagonalization.

---

## 2. Core Challenges

### Challenge 1: State Identification at High Energy

**Problem:** States like \|030⟩, \|130⟩, \|040⟩ can hybridize. Label strings from `edict` become unreliable.

**Solution:** Use number operator expectation values:
```
⟨N_Q0⟩, ⟨N_DTC⟩, ⟨N_Q1⟩ = ⟨ψ_k|N̂|ψ_k⟩
```

Find state where expectation values match target (e.g., \|130⟩ → ⟨N⟩ ≈ (1, 3, 0)).

### Challenge 2: Avoided Crossings

**Problem:** Near DTC state crossings:
- Labels swap between eigenstates
- Two distinct eigenstates coexist
- Each gives different χ value

**Physical reality:**
- At crossing, two hybridized eigenstates exist: |ψ₊⟩ and |ψ₋⟩
- Qubit spectroscopy shows TWO peaks (one per eigenstate)
- As flux sweeps through, one peak grows, other shrinks
- No discontinuity in individual branches — labels swap, physics doesn't

**Solution:** Track BOTH branches through crossings.

### Challenge 3: Validation

**Problem:** How to know if calculated χ is trustworthy?

**Solution:** Track purity metrics:
- State purity: |⟨bare|dressed⟩|²
- Identification error: distance from target ⟨N⟩ values
- Flag points near crossings

---

## 3. Required Modifications to `simOne()`

Current `simOne(..., return_full=True)` returns:
```python
{
    'edict': dict,
    'eigenvectors': ndarray,
    'eigenvalues': ndarray,
    'resonator_Q_ops': dict,
    'state_labels_ordered': list,
    'element_order': list,
    'hilbert_dim': int,
    'sort_dict': dict,
}
```

**Missing:** Number operators `Ns` (built internally but not returned).

### Option A: Modify `simOne()` in `parallelsimulator.py`

Add to `return_full` dict:
```python
'number_operators': Ns,  # List of [N_Q0, N_DTC, N_Q1, ...] in full Hilbert space
```

### Option B: Create wrapper `simOne_extended()` in new module

Duplicate relevant logic, return number operators. Avoids modifying core code.

**Recommendation:** Option A (cleaner), but Option B acceptable if avoiding core changes.

---

## 4. Module Structure

### File: `dispersive_coupler.py`

```
dispersive_coupler.py
├── State Identification
│   ├── find_state_by_excitations()
│   ├── find_states_near_excitation()
│   └── get_state_purity()
├── χ Calculation
│   ├── compute_chi_for_state()
│   ├── get_chi_both_branches()
│   └── get_chi_with_validation()
├── Flux Sweeps
│   ├── sweep_chi_single_branch()
│   ├── sweep_chi_both_branches()
│   └── track_states_adiabatic()
└── Analysis Utilities
    ├── detect_crossings()
    ├── get_crossing_gap()
    └── summarize_chi_sweep()
```

---

## 5. Function Specifications

### 5.1 State Identification

```python
def find_state_by_excitations(
    eigenvectors: np.ndarray,
    number_operators: List[np.ndarray],
    target_n: Tuple[int, ...],
    tolerance: float = 0.5
) -> Tuple[int, float]:
    """
    Find eigenstate index matching target excitation numbers.

    Parameters
    ----------
    eigenvectors : ndarray, shape (hilbert_dim, n_states)
        Columns are eigenstates in bare product basis
    number_operators : list of ndarray
        [N_Q0, N_DTC, N_Q1] in full Hilbert space
    target_n : tuple
        Target excitation numbers, e.g., (1, 3, 0) for |130⟩
    tolerance : float
        Maximum acceptable error

    Returns
    -------
    state_idx : int
        Index of best-matching eigenstate
    error : float
        Sum of squared deviations from target: Σ(⟨N_i⟩ - target_i)²

    Example
    -------
    >>> idx, err = find_state_by_excitations(V, Ns, (1, 3, 0))
    >>> print(f"State |130⟩ is eigenstate {idx}, error={err:.3f}")
    """
```

```python
def find_states_near_excitation(
    eigenvectors: np.ndarray,
    number_operators: List[np.ndarray],
    target_n: Tuple[int, ...],
    n_candidates: int = 2
) -> List[Tuple[int, float]]:
    """
    Find multiple eigenstates closest to target excitation.

    Use for tracking both branches at avoided crossings.

    Returns
    -------
    candidates : list of (state_idx, error)
        Sorted by error (best match first)
    """
```

```python
def get_state_purity(
    eigenvectors: np.ndarray,
    state_idx: int,
    number_operators: List[np.ndarray],
    target_n: Tuple[int, ...]
) -> dict:
    """
    Compute purity metrics for state identification validation.

    Returns
    -------
    metrics : dict
        'exp_values': tuple of ⟨N_i⟩ for each subsystem
        'error': sum of squared deviations from target
        'max_overlap': |⟨dominant_bare|dressed⟩|²
        'is_pure': bool, True if max_overlap > 0.7
    """
```

### 5.2 χ Calculation

```python
def compute_chi_for_state(
    eigenvalues: np.ndarray,
    eigenvectors: np.ndarray,
    number_operators: List[np.ndarray],
    dtc_state_idx: int,
    q_idx: int = 0,
    dtc_idx: int = 1
) -> float:
    """
    Compute χ for specific DTC eigenstate.

    χ = E_|1,k,0⟩ - E_|0,k,0⟩ - E_|1,0,0⟩ + E_|0,0,0⟩

    Parameters
    ----------
    eigenvalues : ndarray
        Energy eigenvalues
    eigenvectors : ndarray
        Eigenvector matrix
    number_operators : list
        Number operators for state identification
    dtc_state_idx : int
        Which DTC excitation level (3, 4, 5 for 2-excitation manifold)
    q_idx : int
        Qubit position in state tuple (default 0)
    dtc_idx : int
        DTC position in state tuple (default 1)

    Returns
    -------
    chi : float
        Dispersive shift in GHz (multiply by 1000 for MHz)
    """
```

```python
def get_chi_both_branches(
    eigenvalues: np.ndarray,
    eigenvectors: np.ndarray,
    number_operators: List[np.ndarray],
    target_dtc_excitation: int = 3
) -> dict:
    """
    Compute χ for both branches near a DTC excitation level.

    Use when target DTC state may be involved in avoided crossing.

    Returns
    -------
    result : dict
        'chi_lower': float, lower χ value
        'chi_upper': float, upper χ value
        'gap': float, |χ_upper - χ_lower|
        'idx_lower': int, eigenstate index for lower branch
        'idx_upper': int, eigenstate index for upper branch
        'purity_lower': float
        'purity_upper': float
    """
```

```python
def get_chi_with_validation(
    result: dict,
    target_n: Tuple[int, ...],
    purity_threshold: float = 0.7,
    error_threshold: float = 0.3
) -> dict:
    """
    Compute χ with validation metrics.

    Parameters
    ----------
    result : dict
        Output from simOne(..., return_full=True) with number_operators added
    target_n : tuple
        Target state, e.g., (1, 3, 0)
    purity_threshold : float
        Minimum purity to consider valid
    error_threshold : float
        Maximum identification error to consider valid

    Returns
    -------
    output : dict
        'chi_MHz': float
        'state_idx': int
        'exp_values': tuple of ⟨N⟩
        'error': float
        'purity': float
        'is_valid': bool
        'warning': str or None
    """
```

### 5.3 Flux Sweeps

```python
def sweep_chi_both_branches(
    simDict: dict,
    phi_values: np.ndarray,
    dtc_excitations: List[int] = [3, 4, 5],
    coupler_name: str = 'DT1',
    parallel: bool = False,
    processes: int = 4
) -> dict:
    """
    Sweep χ vs flux, tracking both branches at crossings.

    Parameters
    ----------
    simDict : dict
        Base simulation parameters
    phi_values : ndarray
        Flux values to sweep (Φ/Φ₀)
    dtc_excitations : list
        DTC eigenstate indices to track (e.g., [3,4,5] for 2-excitation manifold)
    coupler_name : str
        Key in simDict['transmons'] for the DTC
    parallel : bool
        Use multiprocessing
    processes : int
        Number of parallel processes

    Returns
    -------
    results : dict
        'phi': ndarray of flux values
        'chi_k_lower': ndarray for each k in dtc_excitations
        'chi_k_upper': ndarray for each k in dtc_excitations
        'gap_k': ndarray of |χ_upper - χ_lower| for each k
        'valid_k': ndarray of bools for each k
        'crossings': list of detected crossing locations
    """
```

```python
def track_states_adiabatic(
    simDict: dict,
    phi_values: np.ndarray,
    target_states: Dict[str, Tuple[int, ...]],
    ref_idx: int = None,
    coupler_name: str = 'DT1'
) -> dict:
    """
    Track specific states through flux sweep using eigenvector overlap.

    Follows adiabatic branches — no label swapping artifacts.

    Parameters
    ----------
    target_states : dict
        {'label': (n_Q0, n_DTC, n_Q1), ...}
        e.g., {'030': (0,3,0), '130': (1,3,0), '040': (0,4,0)}
    ref_idx : int
        Reference flux index for initial state identification.
        Default: middle of sweep (away from edges where crossings often occur)

    Returns
    -------
    tracking : dict
        'phi': flux values
        'state_indices': {label: [idx at each phi]}
        'energies': {label: [E at each phi]}
        'purities': {label: [purity at each phi]}
    """
```

### 5.4 Analysis Utilities

```python
def detect_crossings(
    phi_values: np.ndarray,
    chi_lower: np.ndarray,
    chi_upper: np.ndarray,
    gap_threshold_MHz: float = 0.5
) -> List[dict]:
    """
    Detect avoided crossings from χ sweep data.

    Returns
    -------
    crossings : list of dict
        Each: {'phi': float, 'gap_MHz': float, 'idx': int}
    """
```

```python
def summarize_chi_sweep(results: dict) -> str:
    """
    Generate text summary of χ sweep results.

    Includes:
    - χ range for each DTC state
    - Crossing locations and gaps
    - Validity statistics
    """
```

---

## 6. Required Changes to Existing Code

### 6.1 Modify `parallelsimulator.py`

In `simOne()`, add to the `return_full` block:

```python
if return_full:
    return {
        'edict': edict,
        'eigenvectors': v,
        'eigenvalues': np.real(w),
        'resonator_Q_ops': resonator_Q_ops,
        'state_labels_ordered': state_labels_ordered,
        'element_order': element_order,
        'hilbert_dim': len(w),
        'sort_dict': built_sort_dict,
        'number_operators': Ns,  # <-- ADD THIS LINE
    }
```

The `Ns` list is already built earlier in `simOne()` — just needs to be included in return.

### 6.2 Update `__init__.py`

Add exports:

```python
from .dispersive_coupler import (
    find_state_by_excitations,
    find_states_near_excitation,
    get_state_purity,
    compute_chi_for_state,
    get_chi_both_branches,
    get_chi_with_validation,
    sweep_chi_both_branches,
    track_states_adiabatic,
    detect_crossings,
    summarize_chi_sweep,
)
```

---

## 7. Expected Output Examples

### Single Point Calculation

```python
>>> result = simOne(simDict, return_full=True)
>>> chi_data = get_chi_with_validation(result, target_n=(1, 3, 0))
>>> print(chi_data)
{
    'chi_MHz': -2.34,
    'state_idx': 47,
    'exp_values': (0.98, 2.94, 0.02),
    'error': 0.04,
    'purity': 0.91,
    'is_valid': True,
    'warning': None
}
```

### Flux Sweep Output

```
Φ/Φ₀  | χ³_lo | χ³_hi | gap³ | valid | χ⁴_lo | χ⁴_hi | gap⁴ | valid
------|-------|-------|------|-------|-------|-------|------|------
0.20  | -3.2  | -1.8  | 1.4  |  ✓    | -2.1  | -0.9  | 1.2  |  ✓
0.25  | -2.9  | -2.1  | 0.8  |  ✓    | -1.8  | -1.2  | 0.6  |  ✓
0.30  | -2.6  | -2.4  | 0.2  |  ⚠️   | -1.5  | -1.4  | 0.1  |  ⚠️  ← crossing
0.35  | -2.3  | -2.7  | 0.4  |  ✓    | -1.2  | -1.7  | 0.5  |  ✓
0.40  | -1.9  | -3.1  | 1.2  |  ✓    | -0.9  | -2.0  | 1.1  |  ✓

Detected crossings:
  DTC=3 ↔ DTC=?: Φ = 0.30, gap = 0.2 MHz
  DTC=4 ↔ DTC=?: Φ = 0.30, gap = 0.1 MHz
```

### Crossing Plot (conceptual)

```
χ (MHz)
   ↑
   |     ╲        DTC=4 upper branch
-1 |      ╲  ╱
   |       ╲╱     ← crossing at Φ = 0.30
-2 |       ╱╲
   |      ╱  ╲    DTC=3 lower branch
-3 |     ╱
   └──────────────→ Φ/Φ₀
        0.2  0.3  0.4
```

---

## 8. Testing Plan

### Unit Tests

1. **State identification:** Create mock eigenvectors with known ⟨N⟩, verify correct state found
2. **χ calculation:** Compare to manual edict lookup for simple cases
3. **Crossing detection:** Verify crossings detected at known locations

### Integration Tests

1. Run on existing Q-DTC-Q simDict
2. Compare χ values to previous manual calculations
3. Verify smooth branches through known crossings

### Validation

1. Check purity metrics flag crossings correctly
2. Verify adiabatic tracking maintains continuous branches
3. Compare single-branch vs both-branches at non-crossing points (should match)

---

## 9. File Locations

| File | Location |
|------|----------|
| New module | `/mnt/project/dispersive_coupler.py` |
| Modify | `/mnt/project/parallelsimulator.py` (add Ns to return_full) |
| Modify | `/mnt/project/__init__.py` (add exports) |
| Reference | `/mnt/project/rockbottom.py` (DTC construction) |
| Reference | `/mnt/project/analysis.py` (existing χ functions) |

---

## 10. Summary

| Component | Purpose |
|-----------|---------|
| Number operator tracking | Robust state ID at crossings |
| Both-branch calculation | Capture physics at avoided crossings |
| Purity metrics | Validate χ reliability |
| Adiabatic tracking | Smooth flux sweeps without label artifacts |

**Key insight:** At DTC avoided crossings, two eigenstates coexist with distinct χ values. Track both branches for complete physical picture.

---

## Document Info

- **Created:** January 2026
- **Context:** EnhancedSmarterThanARock package extension
- **Target:** Claude Code implementation
