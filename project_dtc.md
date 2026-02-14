# DTC Simulation Project

Double Transmon Coupler simulation, spectroscopy prediction, and dispersive shift calculations.

## Code Repos

- **Simulator**: `landsman2:/home/US8J4928/repos/EnhancedSmarterThanARock/` (always use for simulations)
- **Dispersive shift calculator**: `landsman2:/home/US8J4928/repos/dispersive_shift_calculator/`
- **Spectroscopy prediction**: local `~/repos/dtc_spectroscopy_prediction/`
- **Simulation outputs**: `~/repos/sim_outputs/` (spectroscopy, dtc_plots, dtc_cz_sim_results, etc.)

## Key Docs (in ~/.claude/docs/)

Start with `concepts/quantum_simulation_concepts.md` for refresher.

- `concepts/rockbottom_gotchas.md` - **CRITICAL**: `state_labels_ordered` is NOT wavefunction composition
- `concepts/dtc_state_labeling.md` - energy-ordered index vs physical photon count
- `concepts/transmon_dispersive_purcell_derivation.md` - dispersive shift & Purcell theory
- `simulation_tools/spectroscopy_prediction_workflow.md` - M = V†QV transformation
- `simulation_tools/coupling_transition_matrix_viz.md` - H_c vs Q_qubit vs Q_resonator
- `implementation_plans/dispersive_coupler_plan.md` - χ calculation with both-branch tracking
- `implementation_plans/adiabatic_tracking_plan.md` - DTC transition spectrum with chi-colored traces
- `implementation_plans/precompute_composition_plan.md` - precomputed state composition
- `benchmarks/parallel_simulation_benchmarks.md`
- `debugging_notes/chi_sweep_debugging.md`
- `dtc_cz/dtc_cz_sim_readme.md` - CZ gate simulation
- `dtc_cz/dtc_cz_sim_roadmap.md`

## Default Parameters (Big Endeavour Q-DTC-Q)

```python
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
```

## Compute

Run simulations on **landsman2** (28 cores, 1TB RAM):
- Q-DTC-Q (432 dim): 28 cores, ~0.07s/point
- R-Q-DTC-Q-R (3888 dim): 21 cores, ~11s/point (28 cores SLOWER due to NUMA)
