# Integration Guide: dispersive_coupler.py

## Important Terminology Note

**The state labels (e.g., |0,2,0>) use ENERGY-ORDERED EIGENSTATE INDICES, not physical photon counts.**

When you see a state labeled |0,n,0> in this code:
- The 'n' is the **energy-ordered eigenstate index** of the pre-diagonalized DTC
- It is NOT the physical photon count (bare junction excitations)
- The rockbottom simulator returns `N = np.diag([0,1,2,3,...])` which is just the energy level index

For details on this distinction, see `~/.claude/docs/dtc_simulation_notes.md`.

---

## Quick Start

1. **Copy `dispersive_coupler.py` to your project directory** alongside `rockbottom.py`

2. **Update `__init__.py`** - Add these imports:

```python
# Dispersive shift for Q-DTC coupling
from .dispersive_coupler import (
    simOne_extended,
    get_chi_coupler,
    get_chi_coupler_both_qubits,
    sweep_chi_vs_flux,
    sweep_chi_multiple_levels,
    find_state_by_occupation,
    identify_all_states,
    analyze_coupler_dispersive_shift,
    print_state_table,
    check_state_hybridization,
)
```

---

## Key Functions

### `simOne_extended(simDict)`
Extended simulation returning number operators for robust state ID.

```python
result = simOne_extended(simDict)
# Returns: edict, eigenvectors, eigenvalues, number_operators, element_order
```

### `get_chi_coupler(result, coupler_excitation=2)`
Calculate chi_{Q-DTC}^{(n)} using robust state identification.

```python
chi_2 = get_chi_coupler(result, coupler_excitation=2)  # GHz, n=2 energy level
print(f"chi^(2) = {chi_2 * 1000:.2f} MHz")
```

### `sweep_chi_vs_flux(simDict, phi_values, ...)`
Sweep dispersive shift across flux.

```python
data = sweep_chi_vs_flux(simDict, np.linspace(0, 0.5, 51))
plt.plot(data['phi'], data['chi_MHz'])
```

### `find_state_by_occupation(V, Ns, target)`
**Core innovation:** Find states by <N> instead of labels.

```python
# Find state with DTC at energy level 2 even at avoided crossings
idx, err, actual = find_state_by_occupation(V, Ns, [0, 2, 0])
```

---

## Why Robust State ID Matters

At avoided crossings, states hybridize:
- Labels like "0,2,0" can swap between eigenstates
- `edict` lookup gives wrong energies

Solution: Use number operator expectation values
- `<psi|N_DTC|psi> ~ 2` -> this is the state with DTC energy level 2
- Works even when state has 10-20% admixture of other states

**Note:** This identifies states by their energy-ordered index, not physical photon content.
For physical photon counting, you would need to implement tracking of the bare junction
number operators through the DTC diagonalization (see dtc_simulation_notes.md).

---

## Example: Full Analysis

```python
from dispersive_coupler import analyze_coupler_dispersive_shift

analysis = analyze_coupler_dispersive_shift(
    simDict,
    phi=0.25,
    verbose=True
)

# Output:
# chi^(1) = 0.047 MHz (DTC level 1)
# chi^(2) = 0.096 MHz (DTC level 2)
# State purity:
#   |0,0,0>: error=0.000 +
#   |0,1,0>: error=0.001 +
#   |0,2,0>: error=0.003 +
```
