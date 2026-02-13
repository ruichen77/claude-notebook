# Chi Sweep Simulation - Debugging Notes

## Summary

We attempted to create a two-panel chi vs flux plot for the Q0-DTC-Q1 system with:
- **Top panel**: Symlog scale to show full range including divergences at crossings
- **Bottom panel**: Linear scale zoomed to useful operating region (±50 MHz)

## Key Findings

### 1. Flux Range Matters

**Working range:** `np.linspace(-0.5, 0.5, 101)`
**Problematic range:** `np.linspace(0.0, 1.0, 101)`

The simulation hangs when using phi from 0 to 1, but completes successfully with phi from -0.5 to 0.5. Physically these are equivalent (flux is periodic with period Φ₀), but numerically phi=0 and phi=1 may hit edge cases.

### 2. Performance Issue with `get_chi_coupler`

The `get_chi_coupler` function is slow because it calls `find_state_by_occupation` 4 times per call, and each search loops through all ~432 eigenstates doing matrix operations.

**Current script does per flux point:**
- 3 DTC levels × 2 qubits = 6 `get_chi_coupler` calls
- Each call does 4 state searches = 24 `find_state_by_occupation` calls
- Each search: 432 eigenstates × matrix operations

**Previous working script did:**
- 8 `find_state_by_occupation` calls per flux point (3x less work)

### 3. Successful Previous Simulation

The energy spectrum plots were generated successfully with this approach:
```python
phi_values = np.linspace(-0.5, 0.5, 101)

for i, phi in enumerate(phi_values):
    sim = copy.deepcopy(simDict)
    sim['transmons']['DT1']['phi'] = phi
    result = simOne_extended(sim)

    # Track states using find_state_by_occupation (8 calls per point)
    for label, target in [('0,0,0', [0,0,0]), ('0,1,0', [0,1,0]), ...]:
        idx, err, actual = find_state_by_occupation(V, Ns, target, tolerance=1.0)
```

This completed in ~10 minutes for 101 points.

## Files

### On landsman2

- **Script:** `/home/US8J4928/repos/dispersive_shift_calculator/plot_chi_sweep.py`
- **Log:** `/home/US8J4928/repos/dispersive_shift_calculator/chi_sweep.log`
- **Output dir:** `/home/US8J4928/repos/dispersive_shift_calculator/outputs/`

### Existing successful outputs (generated earlier)
- `energy_spectrum_full_sweep.png` - Full spectrum 0-14 GHz
- `purity_vs_flux.png` - State purity showing crossings
- `chi_vs_flux.png`, `chi_vs_flux_full.png`, `chi_vs_flux_zoomed.png`

## Recommended Next Steps

1. **Option A: Optimize `get_chi_coupler`**
   - Use `use_robust_id=False` to use edict labels (faster but may fail at crossings)
   - Or compute all chi values in a single pass through eigenstates

2. **Option B: Reduce computation**
   - Use fewer flux points (e.g., 51 instead of 101)
   - Only compute for DTC levels 1-2 instead of 1-3

3. **Option C: Alternative approach**
   - Compute chi directly from energies stored during the flux sweep
   - Adapt the working energy spectrum code to also output chi

## Script Template (Current Version)

```python
#!/usr/bin/env python3
import numpy as np
import matplotlib
matplotlib.use('Agg')  # Headless backend - must be before pyplot
import matplotlib.pyplot as plt
import sys
import copy
sys.path.insert(0, '/mnt/project')

from dispersive_coupler import simOne_extended, get_chi_coupler

simDict = {
    'transmons': {
        'Q0': {'f01': 4.5, 'anharm': 0.21, 'ng': 0.0, 'cutoff': 6},
        'DT1': {'f01': 3.8, 'anharm': 0.08, 'g': 0.7, 'phi': 0.4, 'ng': 0.0, 'cutoff': 12},
        'Q1': {'f01': 5.0, 'anharm': 0.21, 'ng': 0.0, 'cutoff': 6}
    },
    'couplings': [
        ['Q0', 'DT1;0', 0.18],
        ['Q1', 'DT1;1', 0.18],
        ['DT1;0', 'DT1;1', 0.01],
    ],
    'N': 12,
    'cutoff': 5
}

# USE THIS RANGE (not 0 to 1!)
phi_values = np.linspace(-0.5, 0.5, 101)
```

## Important Reminders

- Always use `matplotlib.use('Agg')` before importing pyplot for headless servers
- Use `sys.stdout.flush()` or `python3 -u` for unbuffered output in tmux
- Use `ssh -T` to suppress pseudo-terminal warnings
- The "DTC level" refers to energy-ordered eigenstate index, NOT physical photon count
