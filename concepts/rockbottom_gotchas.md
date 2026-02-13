# Rockbottom / SmarterThanARock Gotchas

> **See also:** [[DTC State Labeling]], [[Quantum Simulation Concepts]], [[Dispersive Coupler Implementation Plan]]

## CRITICAL: state_labels_ordered is NOT the wavefunction composition!

When using `simOne(simDict, return_full=True)`, the returned dictionary contains:

```python
result = simOne(simDict, return_full=True)
result['state_labels_ordered']  # e.g., ['000', '010', '020', '100', ...]
```

**WARNING**: `state_labels_ordered` is just the **BARE BASIS STATE LABEL** sorted by energy eigenvalue. 
It is **NOT** the actual wavefunction composition!

For hybridized states (especially at higher excitation manifolds where states mix significantly), 
this label can be very misleading. The actual eigenstate may be a superposition of many bare states.

### Correct way to get wavefunction composition:

Use the eigenvector matrix to compute the actual state composition:

```python
def flat_index_to_occupations(idx, dims):
    """Convert flat basis index to occupation tuple."""
    occupations = []
    for d in reversed(dims):
        occupations.append(idx % d)
        idx //= d
    return tuple(reversed(occupations))

def get_state_composition(eigenvectors, state_idx, dims, n_top=3):
    """
    Get the top N components of an eigenstate in the product basis.
    
    Parameters
    ----------
    eigenvectors : ndarray
        V matrix from result['eigenvectors'] (columns = eigenstates)
    state_idx : int  
        Index of the eigenstate to analyze
    dims : list
        Dimensions of each subsystem, e.g., [6, 12, 6] for Q0(6), DTC(12), Q1(6)
    n_top : int
        Number of top components to return
    
    Returns
    -------
    composition : list of (occupation_tuple, probability)
        Top components sorted by probability (descending)
    """
    psi = eigenvectors[:, state_idx]
    probs = np.abs(psi)**2
    
    # Get indices of top N probabilities
    top_indices = np.argsort(probs)[-n_top:][::-1]
    
    composition = []
    for flat_idx in top_indices:
        occ = flat_index_to_occupations(flat_idx, dims)
        prob = probs[flat_idx]
        if prob > 0.01:  # Only include components > 1%
            composition.append((occ, float(prob)))
    
    return composition

# Example usage for Q-DTC-Q system (dims = [6, 12, 6])
result = simOne(simDict, return_full=True)
V = result['eigenvectors']
dims = [6, 12, 6]  # Q0 cutoff, DTC cutoff, Q1 cutoff

# Get composition of state 20
comp = get_state_composition(V, state_idx=20, dims=dims, n_top=4)
for occ, prob in comp:
    print(f"  {prob*100:5.1f}% |{occ[0]},{occ[1]},{occ[2]}⟩")
```

### Example output showing hybridization:

For a highly hybridized state in the 3rd manifold:
```
Q-DTC-Q State 20:
   45.2% |1,1,0⟩   # 45% is Q0=1, DTC=1, Q1=0
   32.1% |0,2,0⟩   # 32% is Q0=0, DTC=2, Q1=0  
   15.3% |0,1,1⟩   # 15% is Q0=0, DTC=1, Q1=1
```

This shows the state is a superposition - NOT a pure |1,1,0⟩ state as `state_labels_ordered` might suggest!

### System dimensions reference:

- **Q-DTC-Q**: `dims = [cutoff_Q0, cutoff_DTC, cutoff_Q1]` (e.g., [6, 12, 6] → 432 dim)
- **R-Q-DTC-Q-R**: `dims = [cutoff_Q0, cutoff_DTC, cutoff_Q1, n_levels_R0, n_levels_R1]` (e.g., [6, 12, 6, 3, 3] → 3888 dim)

The element order follows the order they appear in `simDict['transmons']` then `simDict['resonators']`.
