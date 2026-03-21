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

### Landsman Servers (Direct SSH key access, no 2FA)

**`/data/rzhao/` is a shared 30T NFS mount visible from ALL landsman servers.** Store everything here — repos, scripts, simulation results. Never use `/home/US8J4928/` for simulation work. Because NFS is shared, code and data written on one server are immediately accessible from any other.

**Maximize throughput**: We have 4 online servers totalling 368 vCPUs. For large parameter sweeps, **split the work across multiple servers in parallel** (e.g., split a 400-point sweep into chunks and run simultaneously on landsman4 + landsman5). Always use the Server Selection Protocol to pick the best server(s).

**Full inventory**: See memory file `reference_landsman_servers.md` for detailed hardware specs.

| Server | vCPUs | RAM | Max Workers | Status |
|--------|-------|-----|-------------|--------|
| landsman1 | — | — | — | Decommissioned (broken hardware) |
| landsman2 | 28 | 1.0 TiB | 21 | Online |
| landsman3 | 28 | 502 GiB | 21 | Online |
| landsman4 | 56 | 503 GiB | ~42 (needs benchmarking) | Online |
| landsman5 | 256 | 1.0 TiB | ~192 (needs benchmarking) | Online |

**Key repos on landsman servers:**
- **EnhancedSmarterThanARock**: `/data/rzhao/repos/EnhancedSmarterThanARock/` (always use for simulations)
- **Dispersive shift calculator**: `/data/rzhao/repos/dispersive_shift_calculator/`

**Parallelism note**: On landsman2/3, always use **21 workers** (28 causes NUMA cross-socket contention and can hang eigenbasis computation). On landsman4, use **~42 workers** (56 vCPUs with HT, needs benchmarking). Q-DTC-Q → ~0.07s/point; R-Q-DTC-Q-R → ~11s/point.

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

- **Parallelize with subagents**: When a task has independent subtasks (e.g., probing multiple servers, writing a script while reading docs, analyzing multiple datasets), launch subagents in parallel rather than doing things sequentially. This significantly speeds up multi-step work. Use subagents for anything that takes more than a few seconds and doesn't depend on another task's output. Don't use them for trivial single-tool operations (one grep, one file read).
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

### Server Selection Protocol (MANDATORY before every simulation launch)
**Do NOT hardcode a server.** Before launching any simulation, follow these steps:

**Step 1 — Probe all online servers in parallel:**
```bash
ssh -o ConnectTimeout=5 -T landsman2 "uptime; nproc; free -h | head -2; tmux ls 2>/dev/null || echo 'no tmux sessions'"
ssh -o ConnectTimeout=5 -T landsman3 "uptime; nproc; free -h | head -2; tmux ls 2>/dev/null || echo 'no tmux sessions'"
ssh -o ConnectTimeout=5 -T landsman4 "uptime; nproc; free -h | head -2; tmux ls 2>/dev/null || echo 'no tmux sessions'"
ssh -o ConnectTimeout=5 -T landsman5 "uptime; nproc; free -h | head -2; tmux ls 2>/dev/null || echo 'no tmux sessions'"
```
Skip any server that times out (UNREACHABLE).

**Step 2 — Parse load:** `load_ratio = load_1min / nproc`
- < 0.1 → **idle** (fully available)
- 0.1–0.7 → **partially loaded** (usable, reduce `-j` workers proportionally)
- \> 0.7 → **heavily loaded** (avoid)

Also check `tmux ls` output for running simulation sessions from the catalogue.

**Step 3 — Pick server(s):**
1. **For large sweeps (>100 points): split across multiple idle servers** to maximize throughput. E.g., split 400 points into 200+200 on landsman5 and landsman4, each in its own tmux session writing to the same `/data/rzhao/results/` folder. Combine results after both finish.
2. If single-server is sufficient → landsman5 for large jobs, landsman4 for mid-size, landsman2/3 for smaller jobs
3. If none idle → least loaded, with reduced `-j` workers
4. If all heavily loaded or unreachable → ask the user

**Step 4 — Record decision** in the catalogue entry's `Server selection` field (see template below).

**Important — shared NFS**: `/data/rzhao/` is mounted on every landsman server. All repos, scripts, and results MUST live there (under `/data/rzhao/repos/` and `/data/rzhao/results/`). This is what enables multi-server parallelism — any server can read/write the same files. Never use `/home/US8J4928/`.

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
- **Server**: which machine (landsman2, landsman3, landsman5, CCC, timtam, etc.)
- **Server selection**: load probe results and rationale (e.g., "Checked landsman2 (load 18.3/28, busy), landsman3 (load 0.2/28, idle). Picked landsman3 — idle.")
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
- **Server selection**: Checked landsman2 (load 18.3/28, busy), landsman3 (load 0.2/28, idle), landsman5 (load 0.1/256, idle). Picked landsman5 — idle and most powerful for 200-point sweep.
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
