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
- **Simulation data storage**: **Always use `/data/rzhao/`** for simulation outputs and large data files on any landsman server (shared 30T NFS mount). Do NOT store simulation data under `/home` — it has limited space. Create `/data/rzhao/` if it doesn't exist.
- **EnhancedSmarterThanARock**: `/home/US8J4928/repos/EnhancedSmarterThanARock/` (always use for simulations)
- **Dispersive shift calculator**: `/home/US8J4928/repos/dispersive_shift_calculator/`
- **Hardware**: 28 cores (2×14, no HT), 1 TB RAM, 2 NUMA nodes
- **Parallelism**: Always use **21 workers** for multiprocessing (28 causes NUMA cross-socket contention and can hang eigenbasis computation). Q-DTC-Q → ~0.07s/point; R-Q-DTC-Q-R → ~11s/point.

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

## Notion Knowledge Base

**Workspace**: Quantum Engineering Knowledge Base (Notion, accessible via MCP)

**What goes in Notion** (searchable, shareable, persistent):
- Major findings and results (simulation outcomes, measurement conclusions)
- Device specs and parameters (Big Endeavour, TWPA devices)
- Procedures and how-tos (measurement setup, analysis workflows)
- Sample data and key plots (representative results for reference)
- Project status and roadmaps

**What stays in `~/.claude/docs/`** (Claude Code agent context):
- Claude-specific instructions (CLAUDE.md, project configs)
- Detailed implementation plans (code specs, function signatures)
- Debugging notes (session-specific troubleshooting)
- Code patterns and gotchas (agent working memory)

**Rule**: After completing a significant simulation, measurement, or analysis, offer to save key results to Notion. Use the Projects, Devices, Procedures, and References databases.

---

## Workflow

- **Always use `ssh -T`** for remote commands
- **Use tmux** for simulations >1-2 min: `ssh -T server "tmux new -d -s sim 'cd /path && python -u script.py > output.log 2>&1'"`
- **Always use `python -u`** (unbuffered stdout) when redirecting to log files, so output can be monitored with `tail -f` during long runs
- **Heredoc gotcha**: quotes inside f-strings get stripped. Use scp or write locally first for complex Python.
- **Sessions are disposable, CLAUDE.md is permanent**. Document outcomes in `~/.claude/docs/` and push to git.
- **Docs repo**: `~/.claude/docs/` → `git@github.ibm.com:Ruichen-Zhao/claude_notebook.git`
- **Named CLI sessions** for focused work: `claude --resume "chi-sweep"`, `claude --resume "noise-run"`, etc.

### Simulation Output Convention
**All simulation scripts must save results into timestamped folders:**
- **Folder**: `results/YYYYMMDD_HHMM_<server>_<script_name>/`
- **Data**: `.npz` file with all numerical arrays (reproducible, reloadable)
- **Plots**: `.html` Plotly interactive plots
- **Settings**: `.json` with simulation parameters (optional but recommended)
- Create via `os.makedirs(out_dir, exist_ok=True)` at script start
- After run, `scp` timestamped folder to local `~/projects/<project>/sim_outputs/`

### Simulation Catalogue (MANDATORY)
**Every project has a `catalogue/` directory with weekly log files. You MUST update it for every simulation run.**

This is critical for session recovery — if the computer reboots or the session crashes, the next agent reads the catalogue to know exactly what was running and how to check on it.

**File structure:**
```
~/projects/<project>/catalogue/
├── 2026-W12.md    # Week of 2026-03-16 (Mon) to 2026-03-22 (Sun)
├── 2026-W11.md    # Week of 2026-03-09 to 2026-03-15
└── ...
```
- **Filename**: `YYYY-WNN.md` where NN is the ISO week number (Monday-start)
- Create the file if it doesn't exist when logging a new run

**On session start:**
1. Read the **current week's** catalogue file (and previous week's if today is Mon/Tue)
2. Check status of any `🟡 RUNNING` entries — report to user before proceeding
3. **Offer**: "Need context from earlier weeks? I can check `catalogue/` for older logs."

**BEFORE launching a simulation:**
Append a new entry to the current week's file with ALL of these fields:
- **Run ID**: sequential within the week (e.g., `### W12-01. <descriptive name>`)
- **Status**: `🟡 RUNNING` / `✅ COMPLETED` / `❌ FAILED` / `⏸️ PENDING`
- **Launched**: timestamp (YYYY-MM-DD HH:MM)
- **Server**: which machine (landsman2, CCC, timtam, etc.)
- **tmux session**: session name (e.g., `tmux attach -t sim_name`)
- **Server dir**: full path to results on the remote server
- **Log file**: full path to the log file for monitoring (`tail -f ...`)
- **Script**: what script was run and with what key arguments
- **Config/Parameters**: key simulation parameters (brief)
- **Purpose**: one-line description of what this run is investigating
- **How to check**: exact command to check status (e.g., `ssh -T server "tail -5 /path/to/log"`)
- **Expected duration**: rough estimate if known
- **Local copy**: where results will be scp'd to (leave blank until done)

**AFTER a simulation completes (or fails):**
1. Update the status field (`🟡 RUNNING` → `✅ COMPLETED` or `❌ FAILED`)
2. Add **Results summary**: key findings, output file paths
3. Add **Local copy**: path after scp'ing results locally

**Template:**
```markdown
### W12-01. Chi Sweep Big Endeavour
- **Status**: 🟡 RUNNING
- **Launched**: 2026-03-18 14:30
- **Server**: landsman2
- **tmux session**: `sim_chi_sweep` → `ssh -T landsman2 "tmux attach -t sim_chi_sweep"`
- **Server dir**: `/data/rzhao/results/20260318_1430_landsman2_chi_sweep/`
- **Log file**: `ssh -T landsman2 "tail -20 /data/rzhao/results/.../run.log"`
- **Script**: `python -u sweep_chi.py --phi-range 0.11 0.5 --n-points 200`
- **Config**: Q-DTC-Q, default BE params, phi sweep 0.11–0.5, 200 points
- **Purpose**: Map dispersive shift vs flux for Big Endeavour
- **How to check**: `ssh -T landsman2 "tail -5 /data/rzhao/results/.../run.log"`
- **Expected duration**: ~3 hours (200 pts × ~50s/pt)
- **Local copy**: _(pending)_
```

---

## Diary (Daily Summary)

**At the end of every session** (or when the user says "done for today"), write a diary entry to **Notion**:

- **Where**: Create a page in the Notion workspace, titled `Diary — YYYY-MM-DD — <project>`
- **If multiple sessions in one day**, update the existing day's page (append, don't overwrite)
- **Content**:
  - **Tasks Completed** — brief description of each task done today
  - **Key Conclusions / Findings** — what was learned, decided, or confirmed
  - **Simulation & Data Links** — paths to `sim_outputs/` folders, key files created/modified, with brief descriptions of what each contains
  - **Next Steps** — what remains to be done or follow-up items
- **Proactively offer** to write the diary if the session involved meaningful work (simulations, analysis, debugging) — don't wait to be asked
- Keep entries concise but complete enough to reconstruct context in a future session

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
