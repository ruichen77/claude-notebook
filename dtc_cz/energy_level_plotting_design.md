# Energy Level Plotting — Design Decisions

## Problem

When plotting dressed energy levels vs flux, the original code looked up state
labels (`edict['100']`, `edict['010']`, etc.) independently at each flux point.
At avoided crossings, the labeling algorithm can flip which eigenstate gets a
given label, causing **kinks** (sharp jumps) in the energy traces.

This was the same issue encountered in the dispersive shift calculator work
(see `debugging_notes/chi_sweep_debugging.md`).

## Solution: Energy-Sorted Index + Composition Hover

### Lines: Plot by energy-sorted index (smooth by construction)

Eigenvalues from `np.linalg.eigh` are sorted by energy at every flux point.
Since eigenvalues vary continuously with flux, plotting eigenvalue index `k`
across all flux points produces a smooth, continuous curve — no tracking needed.

This is the same approach used in the spectroscopy prediction code
(`dtc_spectroscopy_prediction/plot_lines_v2.py`), where it was validated
to give smooth traces through all avoided crossings.

### Hover: Bare-state composition (top 3 or top 5)

At each flux point, for each dressed eigenstate `|psi_k>`, compute the
bare-state decomposition: `|<bare_i|psi_k>|^2` for all bare product states.

- If a single bare state has > 50% weight ("dominant"), show top 3 components
  and label the point with that state name (e.g., "|1,0,0> (Q0)")
- If no bare state exceeds 50% ("mixed"), show top 5 components and label
  the point as "mixed"

This lets the user see exactly what physical state each energy level
corresponds to, and how the character evolves through avoided crossings.

### Trace labels/colors: Assigned at idle point

Each trace is named and colored based on its dominant bare-state character
at the **idle flux point** (phi=0.325), where all states are well-separated
and nearly pure. This gives stable, meaningful colors:

- Blue = Q0 character at idle
- Red = DTC character at idle
- Green = Q1 character at idle

As the trace passes through an avoided crossing, the **color stays the same**
(it's the same energy branch) but the **hover text** shows the changing
composition — e.g., the blue "Q0" trace smoothly becomes mixed at the
anticrossing, and the hover reveals the hybridization.

## Alternatives Considered

### Wavefunction overlap tracking (adiabatic continuation)

Available in `state_tracking.track_states_vs_flux(method='overlap')`.
Tracks states by `|<psi(phi_k)|psi(phi_{k+1})>|` between adjacent flux points.
This follows the adiabatic branch, which is useful for dispersive shift
calculations where you must identify a specific state across flux.

**Not used here** because energy-sorted index is simpler and gives the same
smooth curves. The adiabatic approach would also change the trace color
through an avoided crossing (the "Q0" trace would become the "DTC" trace
after the crossing), which is less intuitive for visualizing the spectrum.

### Number operator expectation values (occupation matching)

Available in `dispersive_coupler.py:find_state_by_occupation()`.
Identifies states by `<psi|N_Q0|psi>`, `<psi|N_DTC|psi>`, `<psi|N_Q1|psi>`.
This is what the original code effectively did via `edict` labels.

**Caused the kink problem** because the assignment is done independently
at each flux point and can flip at crossings where two states have similar
occupation numbers.

## Implementation

File: `dtc_cz_sim/examples/layer0_energy_levels.py`

Key functions:
- `get_composition(eigvecs, state_idx)` — bare-state decomposition
- `make_hover_text(composition, energy, phi)` — builds hover with top 3/5
- Compositions precomputed once via `sweep_spectrum()` which returns eigenvectors
- Dominant character threshold: 50% (configurable via `DOMINANCE_THRESHOLD`)

## Related Docs

- `concepts/rockbottom_gotchas.md` — `state_labels_ordered` is NOT composition
- `concepts/dtc_state_labeling.md` — energy-ordered index vs photon count
- `debugging_notes/chi_sweep_debugging.md` — label-swapping at crossings
- `implementation_plans/adiabatic_tracking_plan.md` — wavefunction overlap approach
