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

## CZ Gate Simulation (dtc_cz_sim)

- **Repo**: `landsman2:/home/US8J4928/repos/dtc_cz_sim/`
- **Results**: `landsman2:.../dtc_cz_sim/results/` → copy locally to `~/projects/dtc/sim_outputs/dtc_cz_sim_results/` (see Result Organization below)
- **Roadmap**: `~/.claude/docs/dtc_cz/dtc_cz_sim_roadmap.md`
- **Energy level plotting design**: `~/.claude/docs/dtc_cz/energy_level_plotting_design.md`
- **Key scripts**:
  - `examples/layer0_energy_levels.py` — Energy spectrum + manifold plots (smooth energy-sorted lines, composition hover)
  - `examples/demo_cz_gate.py` — Layer 0+1 full demo (spectrum, ZZ, adiabatic phase, population dynamics, fidelity vs time)
  - `examples/layer0_cz_design.py` — 2D (phi, T_gate) sweep for CZ operating point

### Energy Level Plotting Gotchas
- **Kinks at avoided crossings**: Never label states per-flux-point via `edict`/`find_state_by_occupation`. Plot by energy-sorted index for smooth lines; use hover for state identification.
- **Manifold classification**: Use energy range at sweet spot, NOT digit sum (e.g., `|0,2,0>` has digit sum 2 but is a single DTC excitation in the low band).
- **Sweet spot coloring (phi=0.5)**: States are >97% pure at DTC sweet spot vs ~94% at idle. Always use phi=0.5 as reference for color/name assignment.
- **scp glob quoting**: `scp "landsman2:path/*" dest/` — must quote to prevent local zsh glob expansion.

## Result Organization

**Local results** live in `~/projects/dtc/sim_outputs/dtc_cz_sim_results/`, organized into timestamped folders per run:

```
dtc_cz_sim_results/
├── 20250213_2038_layer0_energy_spectrum/
├── 20250213_2138_demo_cz_gate_layer1/
└── 20250213_2246_fixed_time_cz_comparison/
```

**Naming convention**: `YYYYMMDD_HHMM_<short_summary>/`
- Timestamp = when the sim started on landsman2
- Summary = script name or brief description of what was run

**Workflow**:
1. Sims run on landsman2, outputs land in `landsman2:.../dtc_cz_sim/results/` (flat, gitignored)
2. After a run, `scp` results locally into a new timestamped folder
3. Remote `results/` stays flat (scratch space); local copy is the organized archive

## Compute

Run simulations on **landsman2** (28 cores, 1TB RAM):
- Q-DTC-Q (432 dim): 28 cores, ~0.07s/point
- R-Q-DTC-Q-R (3888 dim): 21 cores, ~11s/point (28 cores SLOWER due to NUMA)
