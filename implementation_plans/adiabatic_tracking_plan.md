# Plan: DTC Transition Spectrum with Chi-Colored Traces

## Goal
Create plots that directly compare to 2D Ramsey experiment heatmaps, showing:
- DTC transition energies vs flux (where you drive)
- Chi values (Ramsey oscillation frequency you'll measure)
- All 4 states involved in chi calculation highlighted

---

## Step 0: Git Checkpoint
Before making changes, commit current code as checkpoint:
```bash
cd /home/US8J4928/repos/dispersive_shift_calculator
git add -A
git commit -m "Checkpoint: working chi/plotting scripts before adiabatic tracking"
git push
```

---

## Implementation Steps

### Step 1: Add Adiabatic Sweep Wrapper
Use existing `track_states_adiabatic()` from `dispersive_coupler.py`.
Add wrapper in `interactive_plots.py`:
```python
def sweep_adiabatic(simDict, phi_values, dtc_levels=[3,4,5]):
    """
    Track states adiabatically for DTC levels.

    For each level n, tracks:
      - |0,0,0⟩ (ground)
      - |1,0,0⟩ (Q0 excited)
      - |0,n,0⟩ (DTC level n)
      - |1,n,0⟩ (Q0 + DTC level n)

    Returns smooth energy tracks + compositions.
    """
```

### Step 2: Compute Derived Quantities
```python
def compute_transition_and_chi(adiabatic_data):
    """
    For each DTC level n:
      - E_transition_n = E_{0,n,0} - E_{0,0,0}  (DTC drive frequency)
      - chi_n = E_{1,n,0} - E_{0,n,0} - E_{1,0,0} + E_{0,0,0}  (Ramsey freq)
    """
```

### Step 3: Create New Plots

#### A. DTC Transition Spectrum (chi-colored)
- **Y-axis**: f_transition (GHz) = E_{0,n,0} - E_{0,0,0}
- **X-axis**: Φ/Φ₀
- **Color**: χ_n value (use colorscale, e.g., diverging blue-white-red)
- **Traces**: One per DTC level (n=3,4,5)
- **Hover**: transition energy, χ, purity, composition

#### B. Energy Level Diagram with Chi States Highlighted
- Show all 4 states for each chi calculation:
  - |0,0,0⟩ black solid (ground)
  - |1,0,0⟩ black dashed (Q0 excited)
  - |0,n,0⟩ colored solid (DTC level n)
  - |1,n,0⟩ colored dashed (Q0 + DTC level n)
- Same color for paired states (green=level 3, purple=level 4, orange=level 5)
- Hover shows composition

#### C. Chi vs Flux (improved)
- Show χ_n for n=3,4,5
- Y-axis zoomed to useful range (±100 MHz)
- Hover shows all 4 energies used in formula

---

## Files to Modify

1. **`/home/US8J4928/repos/dispersive_shift_calculator/interactive_plots.py`**
   - Add `sweep_adiabatic()`
   - Add `compute_transition_and_chi()`
   - Add `create_transition_spectrum_plot()` (HTML + PNG)
   - Update existing plot functions for adiabatic data

2. **`/home/US8J4928/repos/dispersive_shift_calculator/generate_interactive_plot.py`**
   - Use adiabatic sweep
   - Generate all new plot types
   - Keep existing plot types too

---

## Output Files

```
outputs/energy_sweep_YYYYMMDD_HHMMSS/
├── sweep_data.npz                          # Raw data
├── simDict.json                            # Parameters
│
├── interactive_energy_spectrum.html        # Energy levels (adiabatic)
├── interactive_energy_spectrum_zoomed.html
├── energy_spectrum.png
│
├── interactive_transition_spectrum.html    # NEW: DTC transitions colored by chi
├── transition_spectrum.png                 # NEW: PNG version
│
├── interactive_chi.html                    # Chi vs flux
├── chi_with_energies.png                   # Chi + 4 contributing energies
├── chi_vs_flux_zoomed.png
│
└── purity_vs_flux.png                      # Hybridization indicator
```

---

## Visual Design

### Color Scheme for DTC Levels
| Level | Color  | States |
|-------|--------|--------|
| n=3   | Green  | |0,3,0⟩ solid, |1,3,0⟩ dashed |
| n=4   | Purple | |0,4,0⟩ solid, |1,4,0⟩ dashed |
| n=5   | Orange | |0,5,0⟩ solid, |1,5,0⟩ dashed |
| Ground/Q0 | Black | |0,0,0⟩ solid, |1,0,0⟩ dashed |

### Chi Colorscale (for transition spectrum)
- Blue → White → Red (diverging)
- Center (white) at χ = 0
- Saturates at ±50 MHz or ±100 MHz

---

## Verification

1. Energy curves smooth (no jumps) through avoided crossings
2. DTC transition energies match E_{0,n,0} - E_{0,0,0}
3. Chi values match E_{1,n,0} - E_{0,n,0} - E_{1,0,0} + E_{0,0,0}
4. In non-hybridized regions, matches per-point method
5. Overlay on experimental 2D heatmap should align
