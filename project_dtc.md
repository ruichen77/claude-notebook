# DTC Simulation Project

Double Transmon Coupler simulation, spectroscopy prediction, and dispersive shift calculations.

## Code Repos

- **Simulator**: `/data/rzhao/repos/EnhancedSmarterThanARock/` (always use for simulations)
- **Dispersive shift calculator**: `/data/rzhao/repos/dispersive_shift_calculator/`
- **Spectroscopy prediction**: local `~/repos/dtc_spectroscopy_prediction/`
- **qss-pulsegen**: `/data/rzhao/repos/qss-pulsegen/` (IBM pulse shape library, source of Ramen pulse)
- **duffing_cz (qiskit-dynamics)**: `git@github.ibm.com:Ruichen-Zhao/dtc_cz_sim_qiskit.git` — synced via git across all machines. See `dtc_cz/duffing_cz_package.md`.
  - Local (dev): `~/repos/dtc_cz_sim_qiskit/`
  - landsman3: `.../dtc_cz_sim_qiskit/` (qiskit_dyn env)
  - CCC: `/u/ruichenzhao/repos/dtc_cz_sim_qiskit/` (duffing_cz env, A100 GPUs)
- **adiabatic_cz (charge-basis + eigenbasis)**: `git@github.ibm.com:Ruichen-Zhao/dtc_adiabatic_sim.git` — rotating-basis CZ sim. See `dtc_cz/adiabatic_cz_package.md`.
  - Local (dev): `~/repos/dtc_adiabatic_sim/`
  - CCC (GPU): `/u/ruichenzhao/repos/dtc_adiabatic_sim/` (duffing_cz env)
  - landsman3 (CPU): `.../dtc_adiabatic_sim/` (qiskit_dyn env)
- **Simulation outputs**: `~/repos/sim_outputs/` (spectroscopy, dtc_plots, dtc_cz_sim_results, etc.)

### Development Workflow (duffing_cz)
**Develop locally → git push → git pull on remote → run.** Never edit directly on remote.
1. Edit code in `~/repos/dtc_cz_sim_qiskit/`
2. `git commit && git push`
3. On remote: `cd ~/repos/dtc_cz_sim_qiskit && git pull`
4. Run on landsman3: `conda run -n qiskit_dyn python -u ...`
5. Run on CCC: submit via `bsub` (see `ccc/submit.sh`)

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
- `dtc_cz/cz_pulse_design_intuition.md` - **KEY**: Avoided crossing strategies (adiabatic vs diabatic), leakage/seepage taxonomy, pulse shape tradeoffs
- `dtc_cz/qss_pulsegen_pulse_shapes.md` - **Pulse shape reference**: All shapes in qss-pulsegen, Ramen pulse math & physics, mapping to DTC CZ parameters
- `dtc_cz/pulse_shape_comparison.md` - **CRITICAL**: qss-pulsegen vs dtc_cz_sim pulse comparison. Our "trapezoid" uses COSINE ramps (not linear like hardware). Ramen/square_ramen match.
- `dtc_cz/duffing_cz_package.md` - **duffing_cz package**: API reference, usage examples, dev workflow
- `dtc_cz/adiabatic_cz_package.md` - **adiabatic_cz package**: charge-basis + eigenbasis CZ sim, GPU-compatible

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

- **Repo**: `/data/rzhao/repos/dtc_cz_sim/`
- **Results**: `/data/rzhao/repos/dtc_cz_sim/results/` → copy locally to `~/projects/dtc/sim_outputs/dtc_cz_sim_results/`
- **Roadmap**: `~/.claude/docs/dtc_cz/dtc_cz_sim_roadmap.md`
- **Energy level plotting design**: `~/.claude/docs/dtc_cz/energy_level_plotting_design.md`
- **Key scripts**:
  - `examples/systematic_cz_2d.py` — **PRIMARY**: 2D (gate_time × phi_interaction) sweep, π-contour extraction, Layer 1 at ALL 40 contour points
  - `examples/manual_cz_sweep.py` — Fix gate time (100,200,300,1000ns), tune amplitude for π phase
  - `examples/iswap_diagnostic.py` — Residual iSWAP analysis: g_eff(Φ) from single-excitation manifold, 3×3 SWAP prediction (**written, not yet deployed**)
  - `examples/rotating_frame_demo.py` — Rotating frame comparison (5 validation plots)
  - `examples/layer0_energy_levels.py` — Energy spectrum + manifold plots (50 states)
  - `examples/demo_cz_gate.py` — Layer 0+1 full demo (safe-region only, for reference)

### CZ Gate Design Workflow
- **Always fix gate time first**, then tune phi_interaction (pulse amplitude) to hit π phase
- **Pulse shapes**: `trapezoid_pulse` (LINEAR ramps, hw-matching), `cosine_flat_top_pulse` (cosine ramps, best performer), square, and **Ramen** (crossing-aware). See `dtc_cz/pulse_shape_comparison.md` and `dtc_cz/qss_pulsegen_pulse_shapes.md`.
  - **v3 fix (Feb 18)**: `trapezoid_pulse` corrected to LINEAR ramps matching qss-pulsegen hardware. Old cosine version preserved as `cosine_flat_top_pulse`.
  - **v2→v3 consistency**: All 8 Ramen configs produce bit-identical results between v2 and v3. Adaptive ODE solver is fully deterministic.
- **Max excursion**: phi_interaction >= 0.11 (**HARDCODED LIMIT** — see below)

### ⚠️ DANGER ZONE: phi < 0.11 Anticrossing (STATE TRACKER BREAKS)
The |100⟩↔|020⟩ anticrossing at phi ~ 0.094–0.106 breaks ALL dressed-state tracking in the codebase.
**ANY function that tracks dressed states through this region will give WRONG results.**

Known affected functions:
- `sweep_zz()` — ZZ jumps from −20 MHz to +193 MHz (label swap)
- `extract_ramen_params()` — gives spurious delta/det0 if phi_ref < 0.11 **(BURNED US: phi_ref=0.05 gave delta=207 MHz vs correct 252 MHz, made Full Ramen appear 2x better than it is)**
- Any future function that calls `find_state_by_occupation()` or similar state tracking across this crossing

**RULE: Never pass phi < 0.11 to any state-tracking function.** This includes:
- `phi_ref` in `extract_ramen_params()` → use 0.15
- `phi_min` in ZZ sweeps → use 0.11
- `phi_interaction` bounds in brentq → use 0.111 as lower bound
- Any new analysis that sweeps flux → clamp at 0.11

**Workaround**: `phi_min = 0.11` is hardcoded in `spectrum_pulse_overlay.py`. The ZZ interpolator
uses only clean data at phi >= 0.11, and all pulse trajectories are clamped at this limit.
- ZZ at phi=0.11: −24.6 MHz (deepest accessible ZZ)
- Minimum gate times at phi_min: Square 20ns, Trapezoid 34ns, Ramen 75ns
- **This is a HACK**: the true fix is to recompute ZZ with proper adiabatic state tracking through the anticrossing
- Affects: `spectrum_pulse_overlay.py`, any script using `sys2d_results.npz` ZZ data below phi=0.11

### Systematic 2D Sweep (`examples/systematic_cz_2d.py`)
Automated exploration of the full (gate_time, phi_interaction) parameter space:
1. **Pre-compute** ZZ and energy spectrum vs flux (1001 points, 50 states)
2. **Constraint**: max |ZZ| at phi_interaction ≤ 10 MHz (finds the phi boundary)
3. **2D adiabatic sweep**: 40 log-spaced gate times (150–1000 ns) × 50 phi_interaction values → conditional phase map
4. **π-contour extraction**: for each gate time, brentq finds the phi_interaction giving exactly −π phase
5. **Layer 1 at ALL contour points**: runs full 432-dim unitary simulation at all ~40 π-contour points
6. **Detail diagnostics**: `multi_state_dynamics` at 5 evenly-spaced points for conditional phase buildup + population dynamics
7. **Data saving**: `sys2d_settings.json` (parameters) + `sys2d_results.npz` (all numerical arrays)
8. **Outputs** (7 Plotly HTML plots + 2 data files):
   - `sys2d_spectrum_zz.html` — energy spectrum (comp states highlighted, hover composition) + ZZ, shared x-axis
   - `sys2d_phase_map.html` — 2D heatmap of conditional phase / π, π-contour overlay, log-scale gate time axis
   - `sys2d_pi_contour.html` — π-contour: gate time vs phi and vs |ZZ|, with selected points
   - `sys2d_fidelity_leakage.html` — gate error (1−F) and avg leakage at the 5 selected points
   - `sys2d_populations.html` — population dynamics (|11⟩ input) at each selected point
   - `sys2d_pulses.html` — trapezoid pulse shapes for all 5 selected points overlaid

### Plotting Preferences
- **Always combine energy spectrum + ZZ** in a shared-x-axis stacked plot (energy top, ZZ bottom)
- **Always include spectrum + pulse overlay** for every pulse/gate fidelity study: energy levels vs flux (top, with hover composition) + pulse trajectory phi(t) (bottom, time going downward, shared flux x-axis). Script: `examples/spectrum_pulse_overlay.py`
- **spectrum_pulse_overlay.py** now sweeps gate times [80, 160, 320, 640 ns] × 3 shapes (Square, Trapezoid, Ramen), auto-finds phi_int for −π via brentq, plots on normalized time. Uses hardcoded phi_min=0.11.

### Energy Level Plotting Gotchas
- **Kinks at avoided crossings**: Never label states per-flux-point via `edict`/`find_state_by_occupation`. Plot by energy-sorted index for smooth lines; use hover for state identification.
- **Manifold classification**: Use energy range at sweet spot, NOT digit sum (e.g., `|0,2,0>` has digit sum 2 but is a single DTC excitation in the low band).
- **Sweet spot coloring (phi=0.5)**: States are >97% pure at DTC sweet spot vs ~94% at idle. Always use phi=0.5 as reference for color/name assignment.
- **scp glob quoting**: `scp "landsman2:path/*" dest/` — must quote to prevent local zsh glob expansion.

## Result Organization

**Local results** live in `~/projects/dtc/sim_outputs/`, organized by repo:

```
sim_outputs/
├── dtc_cz_sim_results/           # RockBottom-based CZ sim
└── dtc_cz_sim_qiskit/            # Duffing/qiskit-dynamics CZ sim
```

Follows global Simulation Output Convention (see CLAUDE.md). After a run, `scp` the timestamped folder into `~/projects/dtc/sim_outputs/<repo>/`.

### ⚠️ landsman `adiabatic_cz` install note
Package installed in **qiskit_dyn** conda env via `pip install -e .` (editable). Must use `conda activate qiskit_dyn` or `source conda.sh && conda activate qiskit_dyn` before running.

## Compute

Use Server Selection Protocol (global CLAUDE.md) to pick server. DTC-specific timing:
- Q-DTC-Q (432 dim): ~0.07s/point
- R-Q-DTC-Q-R (3888 dim): ~11s/point (use 21 workers max on landsman2/3)
- **⚠️ JAX/LLVM limit**: `max_map_count=65530` on landsman — max 2–3 JIT compilations per process. Use subprocess isolation for sweeps.
