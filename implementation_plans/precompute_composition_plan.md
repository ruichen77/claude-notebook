# Implementation Plan: Precomputed State Composition & Interactive Plots

## Overview

Two-step implementation:
1. **Step 1**: Precompute state compositions in `simOne_extended()` for efficiency
2. **Step 2**: Build interactive Plotly-based energy level diagrams with hover composition

---

## Step 1: Precompute State Composition

### 1.1 Goal

Currently, `find_state_by_occupation()` loops through all 432 eigenstates every time it's called. For a chi calculation needing 4 states x 3 DTC levels x 101 flux points = 1,212 searches, this is slow.

**Solution**: Compute all expectation values and state compositions ONCE per flux point, then use fast array lookups.

### 1.2 What to Precompute

Add to the return dict of `simOne_extended()`:

```python
{
    # Existing fields...
    'edict': edict,
    'eigenvectors': v,
    'eigenvalues': w,
    'number_operators': Ns,
    'element_order': element_order,
    'hilbert_dim': len(w),

    # NEW fields:
    'exp_values': exp_values,        # shape (n_states, n_modes) - average occupations
    'dims': dims,                     # [cutoff_Q0, cutoff_DTC, cutoff_Q1]
    'compositions': compositions,     # list of top components per eigenstate
}
```

### 1.3 Data Structures

#### `exp_values` array
```python
# Shape: (n_states, n_modes) = (432, 3)
exp_values[k, i] = <psi_k|N_i|psi_k>

# Example:
# exp_values[47, :] = [0.03, 2.1, 0.01]  means eigenstate 47 has <N> ~ |0,2,0>
```

#### `dims` list
```python
# Cutoff dimensions for each subsystem
dims = [6, 12, 6]  # [Q0, DTC, Q1]
```

#### `compositions` list
```python
# For each eigenstate, store top N components
# compositions[k] = [(occupation_tuple, probability), ...]

compositions[47] = [
    ((0, 2, 0), 0.847),
    ((0, 3, 0), 0.089),
    ((1, 2, 0), 0.031),
    ((0, 1, 0), 0.018),
    ((0, 4, 0), 0.008),
]
```

### 1.4 Helper Functions to Add

```python
def flat_index_to_occupations(idx, dims):
    """Convert flat basis index to (n_Q0, n_DTC, n_Q1) tuple."""
    occupations = []
    for d in reversed(dims):
        occupations.append(idx % d)
        idx //= d
    return tuple(reversed(occupations))


def occupations_to_flat_index(occupations, dims):
    """Convert (n_Q0, n_DTC, n_Q1) tuple to flat basis index."""
    idx = 0
    for i, occ in enumerate(occupations):
        multiplier = 1
        for d in dims[i+1:]:
            multiplier *= d
        idx += occ * multiplier
    return idx


def get_state_composition(eigenvectors, state_idx, dims, n_top=5):
    """
    Get top N components of an eigenstate in the product basis.

    Returns: [(occupation_tuple, probability), ...] sorted by probability
    """
    psi = eigenvectors[:, state_idx]
    probs = np.abs(psi)**2

    top_indices = np.argsort(probs)[-n_top:][::-1]

    composition = []
    for flat_idx in top_indices:
        occ = flat_index_to_occupations(flat_idx, dims)
        prob = probs[flat_idx]
        if prob > 1e-6:  # skip negligible components
            composition.append((occ, prob))

    return composition


def precompute_all_compositions(eigenvectors, dims, n_top=5):
    """Precompute compositions for ALL eigenstates."""
    n_states = eigenvectors.shape[1]
    compositions = []
    for k in range(n_states):
        comp = get_state_composition(eigenvectors, k, dims, n_top)
        compositions.append(comp)
    return compositions
```

### 1.5 Fast State Lookup

```python
def find_state_by_occupation_fast(exp_values, target_occupations):
    """
    Find eigenstate matching target occupations using precomputed exp_values.

    O(n_states) simple array operations instead of O(n_states x n_modes) matrix ops.
    """
    errors = np.sum((exp_values - target_occupations)**2, axis=1)
    best_idx = np.argmin(errors)
    return best_idx, errors[best_idx], exp_values[best_idx]
```

### 1.6 Testing Step 1

Run test to verify:
1. New fields exist in result
2. Fast lookup gives same result as old method
3. Compositions are correctly computed

---

## Step 2: Interactive Plotly Energy Diagram

### 2.1 Goal

Create an interactive HTML plot where:
- Energy levels vs flux are shown as curves
- Hovering shows state composition breakdown
- Crossings are highlighted
- Can zoom/pan

### 2.2 Hover Box Content

On hover over any point:

```
Phi/Phi_0 = 0.18
Energy = 8.07 GHz
Eigenstate index: 47

State Composition:
  |0,3,0>  72.3%  ================....
  |0,4,0>  18.1%  =====...............
  |1,3,0>   5.2%  ==..................
  |0,2,0>   2.8%  =...................
  other     1.6%

<N> = [0.05, 3.21, 0.01]
Purity: 72.3%
```

At crossings (low purity):

```
Phi/Phi_0 = 0.20  WARNING: AVOIDED CROSSING
Energy = 8.31 GHz
Eigenstate index: 52

State Composition:
  |0,4,0>  48.2%  ============........
  |0,3,0>  44.7%  ===========.........
  |1,3,0>   3.9%  =...................
  |0,5,0>   2.1%  =...................

<N> = [0.04, 3.48, 0.01]
Purity: 48.2%  <- HYBRIDIZED
```

### 2.3 Data Collection Function

```python
def sweep_with_composition(simDict, phi_values, states_to_track, coupler_name='DT1'):
    """
    Sweep flux and collect full composition data for interactive plotting.

    Parameters
    ----------
    simDict : dict
        Base simulation parameters
    phi_values : ndarray
        Flux values to sweep
    states_to_track : list of tuple
        Target states, e.g., [(0,3,0), (0,4,0), (0,5,0)]
    coupler_name : str
        DTC key in simDict

    Returns
    -------
    sweep_data : dict
        Contains phi values and per-state data (energies, compositions, etc.)
    """
```

### 2.4 Interactive Plot Function

```python
def create_interactive_energy_plot(sweep_data, title, purity_threshold=0.7, output_file="interactive_spectrum.html"):
    """
    Create interactive Plotly energy level diagram.

    - Main traces for each state
    - Yellow markers at low-purity crossing points
    - Hover text shows full composition
    """
```

---

## Implementation Order

### Phase 1: Core Precomputation

1. Add helper functions to `dispersive_coupler.py`:
   - `flat_index_to_occupations()`
   - `occupations_to_flat_index()`
   - `get_state_composition()`
   - `precompute_all_compositions()`
   - `find_state_by_occupation_fast()`

2. Modify `simOne_extended()` to include precomputed data

3. Run test to verify correctness

4. Benchmark: compare speed of old vs new state lookup

### Phase 2: Interactive Plotting

1. Create new file `interactive_plots.py` with:
   - `sweep_with_composition()`
   - `create_interactive_energy_plot()`

2. Create example script `generate_interactive_plot.py`

3. Run and verify HTML output works

4. Test hovering shows correct composition data

---

## Files to Create/Modify

| File | Action |
|------|--------|
| `dispersive_coupler.py` | Modify - add precomputation |
| `test_precompute.py` | Create - test precomputation |
| `interactive_plots.py` | Create - Plotly plotting functions |
| `generate_interactive_plot.py` | Create - example usage |

---

## Dependencies

- `numpy` (already installed)
- `plotly` (may need to install: `pip install plotly`)

---

## Success Criteria

1. **Step 1 complete when:**
   - `simOne_extended()` returns `exp_values`, `dims`, `compositions`
   - `find_state_by_occupation_fast()` gives same results as old method
   - Measurable speedup in chi sweep (target: 3x faster)

2. **Step 2 complete when:**
   - Interactive HTML file generated
   - Hovering shows state composition with percentages
   - Crossings (low purity) are marked
   - Plot can be zoomed/panned
