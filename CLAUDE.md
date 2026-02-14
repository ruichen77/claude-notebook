# Claude Code Configuration for Ruichen

Proactively offer to document reusable info in `~/.claude/docs/` and push to git. Keep this file concise. Split details into sub-files.

---

## Servers

### 2FA Servers (timtam, narya)

Require SSH ControlMaster: user must SSH manually first, keep terminal open, then Claude can piggyback.

- **timtam** - `ssh timtam` (OpenQ network)
- **narya** - `ssh narya` (OpenQ network)

### timtam Directories
- **fridge36B**: `/nas-data1/systems/fridge36B/`
  - Setup guide: `~/.claude/docs/measurement_folder_setup.md`
  - Current alias: `~/.localrc` defines `current` → active measurement folder
  - Noise measurement: `timtam:/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/` → `python -m noise_measurement run -c config.yaml`

### narya Directories
- **fridge1F Big Endeavour**: `/nas-data1/systems/fridge1F/20250728_E_BE/BE001`

### CCC (Cognitive Computing Cluster)
- **Login**: `ssh -T ruichenzhao@ccc-login1.pok.ibm.com` (SSH key, no 2FA - direct access)
- **VPN required** if off-campus
- **GPUs**: ~800+ A100 (40G/80G) + H100 (80G), 8 per node, 103 nodes
- **Scheduler**: LSF (`bsub`/`bjobs`), queues: `normal` (no time limit), `interactive` (6hr)
- **Home**: `/u/ruichenzhao/` (100 GB)
- **Full guide**: `~/.claude/docs/ccc_gpu_guide.md`

### landsman2
- Direct SSH key access
- **EnhancedSmarterThanARock**: `/home/US8J4928/repos/EnhancedSmarterThanARock/` (always use for simulations)
- **Dispersive shift calculator**: `/home/US8J4928/repos/dispersive_shift_calculator/`
- **Hardware**: 28 cores (2×14, no HT), 1 TB RAM, 2 NUMA nodes
- **Parallelism**: Q-DTC-Q → 28 cores (~0.07s/point); R-Q-DTC-Q-R → 21 cores (~11s/point). 28 cores is SLOWER for R-Q-DTC-Q-R (NUMA cross-socket overhead).

---

## DTC Simulation

### Documentation
All docs in `~/.claude/docs/`. Start with `concepts/quantum_simulation_concepts.md` for refresher.

Key references:
- `concepts/rockbottom_gotchas.md` - **CRITICAL**: `state_labels_ordered` is NOT wavefunction composition
- `concepts/dtc_state_labeling.md` - energy-ordered index vs physical photon count
- `simulation_tools/spectroscopy_prediction_workflow.md` - M = V†QV transformation
- `simulation_tools/coupling_transition_matrix_viz.md` - H_c vs Q_qubit vs Q_resonator
- `implementation_plans/dispersive_coupler_plan.md` - χ calculation with both-branch tracking
- `benchmarks/parallel_simulation_benchmarks.md`
- `debugging_notes/chi_sweep_debugging.md`

### Default Parameters (Big Endeavour Q-DTC-Q)
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

---

## Directory Structure

```
~/projects/          # Claude Code workspaces - cd here to start sessions
├── CLAUDE.md        → ~/.claude/docs/CLAUDE.md (global config)
├── dtc/             # DTC simulation project
│   ├── CLAUDE.md    → ~/.claude/docs/project_dtc.md
│   └── sim_outputs/ # simulation results, plots, data
└── twpa/            # TWPA noise project
    ├── CLAUDE.md    → ~/.claude/docs/project_twpa.md
    └── sim_outputs/ # measurement results, plots

~/repos/             # Code repositories only (git repos, pip-installed tools)
~/.claude/docs/      # All .md docs (git-tracked, backed up on GitHub)
```

- **Always `cd ~/projects/<project>` before launching `claude`** for project-scoped sessions
- **Code repos stay in `~/repos/`** - never add CLAUDE.md inside git repos
- **Project folders hold**: CLAUDE.md (symlink), sim_outputs, scratch files, notebooks
- **New projects**: create `~/projects/<name>/`, add `project_<name>.md` in docs repo, symlink as CLAUDE.md

## Workflow

- **Always use `ssh -T`** for remote commands
- **Use tmux** for simulations >1-2 min: `ssh -T server "tmux new -d -s sim 'cd /path && python script.py > output.log 2>&1'"`
- **Heredoc gotcha**: quotes inside f-strings get stripped. Use scp or write locally first for complex Python.
- **Sessions are disposable, CLAUDE.md is permanent**. Document outcomes in `~/.claude/docs/` and push to git.
- **Docs repo**: `~/.claude/docs/` → `git@github.ibm.com:Ruichen-Zhao/claude_notebook.git`
- **Named CLI sessions** for focused work: `claude --resume "chi-sweep"`, `claude --resume "noise-run"`, etc.

---

## Plot Styling

All plots: **bold black borders**.

```python
# Matplotlib
for spine in ax.spines.values():
    spine.set_linewidth(2)
    spine.set_color("black")

# Plotly
fig.update_layout(
    xaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
    yaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
)
```
