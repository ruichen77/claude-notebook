# Claude Code Configuration for Ruichen

## Meta: How to Use This File

**Claude should be proactive** about offering to document useful info, but keep this file concise.

### What to Document
- Frequently needed info (server access, common workflows)
- Non-obvious quirks that are hard to rediscover
- Stable patterns (won't change next week)

### What NOT to Document
- One-off commands or temporary fixes
- Obvious things
- Rapidly changing details

### Guidelines for Claude
- Offer to add high-value, reusable info: *"This seems useful to remember - want me to add it to CLAUDE.md?"*
- User can always say no
- Keep entries brief (bullet points preferred)
- Periodically suggest cleanup if file grows large
- If needed, split into sub-files in `.claude/` directory (only read when relevant)

---

## Remote Server Access (2FA Servers)

The following servers require 2-factor authentication and cannot be accessed directly by Claude Code. Use the **SSH ControlMaster** method:

### Servers Requiring 2FA
- **timtam** - OpenQ network server
- **narya** - OpenQ network server

### How to Enable Claude Access

1. **In a separate terminal**, SSH into the server manually:
   ```bash
   ssh timtam
   # or
   ssh narya
   ```

2. **Complete the 2FA authentication** (password + verification code)

3. **Keep that terminal open** - this maintains the SSH ControlMaster socket

4. **Now Claude can access the server** via the shared socket

### Why This Works

Your `~/.ssh/config` has ControlMaster configured for `*.watson.ibm.com`:
```
ControlMaster auto
ControlPath ~/.ssh/%r@%h:%p
```

When you SSH manually, it creates a socket at `~/.ssh/`. Claude's subsequent SSH commands reuse this socket without needing to re-authenticate.

### Verification

To check if a socket exists:
```bash
ls -la ~/.ssh/ | grep "@"
```

You should see something like:
```
srw-------  ruichen.zhao_ibm.com@champlaincanal-nat.watson.ibm.com:22
```

---

## Server Directories

### timtam (~labuser home)
- `qss-pulsegen` - Pulse generation library
- `qss-instruments` - Instrument control
- `qic-rta-components` - RTA components
- `rusty_chain` - Rust chain tools
- `qic-network`, `qic-rta-driver`, `qic-rta-utils`
- `qiskit-monitoring`, `qiskit-experiments-internal`

### timtam fridge36B Measurement Folders
- **Location**: `/nas-data1/systems/fridge36B/`
- **Setup guide**: `timtam:/nas-data1/systems/fridge36B/MEASUREMENT_FOLDER_SETUP.md` - Read this when asked to create a new measurement/cooldown folder or run noise measurements
- **Current alias**: `~/.localrc` defines `current` alias pointing to active measurement folder
- **Noise measurement package**: `timtam:/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/` - Use `python -m noise_measurement run -c config.yaml`

### narya fridge1F Big Endeavour Measurement Folder
- **Location**: `/nas-data1/systems/fridge1F/20250728_E_BE/BE001`

### CCC (Cognitive Computing Cluster)
- **Login**: `ssh -T ruichenzhao@ccc-login1.pok.ibm.com` (SSH key auth, no 2FA - Claude can connect directly)
- **VPN required** if off-campus (Cisco AnyConnect)
- **GPUs**: ~800+ NVIDIA A100 (40G/80G) and H100 (80G) across 103 nodes, 8 GPUs per node
- **Scheduler**: LSF (`bsub` to submit, `bjobs` to monitor)
- **Queues**: `normal` (batch, no time limit), `interactive` (6hr limit)
- **Home**: `/u/ruichenzhao/` (100 GB quota)
- **Full GPU guide**: `~/.claude/docs/ccc_gpu_guide.md`

### landsman2
- Can be accessed directly (SSH key auth)
- **EnhancedSmarterThanARock**: `/home/US8J4928/repos/EnhancedSmarterThanARock/` - Quantum circuit simulator (always use this for simulations)
- Dispersive shift calculator: `/home/US8J4928/repos/dispersive_shift_calculator/`

#### Hardware Specs
- **CPU**: 2× Intel Xeon E5-2690 v4 @ 2.60GHz
- **Cores**: 28 total (14 per socket, no hyperthreading)
- **RAM**: 1 TB (1006 GB)
- **NUMA**: 2 nodes (cores 0-13 on socket 1, cores 14-27 on socket 2)

#### Parallel Simulation Guidelines (Benchmarked Feb 2026)
- **Full benchmark results**: `landsman2:~/repos/dispersive_shift_calculator/BENCHMARK_RESULTS.md`
- **Q-DTC-Q (432 dim)**: Use **28 cores** → ~7 sec for 101 flux points
- **R-Q-DTC-Q-R (3888 dim)**: Use **21 cores** → ~19 min for 101 flux points
- ⚠️ **NUMA gotcha**: 28 cores is SLOWER than 21 cores for R-Q-DTC-Q-R due to cross-socket overhead
- **Memory per worker**: ~2.9 GB for R-Q-DTC-Q-R
- **Quick estimates**: Q-DTC-Q ~0.07s/point (28 cores), R-Q-DTC-Q-R ~11s/point (21 cores)

---

## DTC Simulation Documentation

- **Quantum simulation concepts**: `~/.claude/docs/concepts/quantum_simulation_concepts.md` - **START HERE** for refreshing memory on operators, eigenstates, Hamiltonians, basis transformations, and the spectroscopy simulation workflow
- **Spectroscopy prediction workflow**: `~/.claude/docs/simulation_tools/spectroscopy_prediction_workflow.md` - Detailed code walkthrough for M = V†QV transformation
- **Coupling matrix visualization**: `~/.claude/docs/simulation_tools/coupling_transition_matrix_viz.md` - Comparison of H_c vs Q_qubit vs Q_resonator operators
- **State labeling & energy indexing**: `~/.claude/docs/concepts/dtc_state_labeling.md` - Important notes on how `rockbottom` labels DTC states (energy-ordered index vs physical photon count)
- **Dispersive coupler implementation plan**: `~/.claude/docs/implementation_plans/dispersive_coupler_plan.md` - Detailed plan for implementing χ calculation with both-branch tracking at avoided crossings
- **Rockbottom gotchas**: `~/.claude/docs/concepts/rockbottom_gotchas.md` - **CRITICAL**: `state_labels_ordered` is NOT the wavefunction composition! Must use eigenvectors to get actual state composition for hybridized states.
- **Benchmark results**: `~/.claude/docs/benchmarks/parallel_simulation_benchmarks.md`
- **Chi sweep debugging**: `~/.claude/docs/debugging_notes/chi_sweep_debugging.md`

## DTC Simulation Default Parameters (Big Endeavour)

Use these parameters for Q-DTC-Q simulations unless otherwise specified:

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

## Workflow Notes

### Development on Remote Servers
- Claude can read/edit files and run commands via SSH
- No need to push/pull through GitHub for quick iterations
- Commit meaningful checkpoints to git when ready

### SSH Flag
- **Always use `ssh -T`** for all remote commands to suppress "Pseudo-terminal will not be allocated" warnings

### Long-Running Simulations
- **Use tmux** for simulations that may take a while (>1-2 minutes)
- Start a tmux session: `ssh -T server "tmux new -d -s sim 'cd /path && python script.py > output.log 2>&1'"`
- Check progress: `ssh -T server "tmux capture-pane -t sim -p"` or `ssh -T server "tail output.log"`
- This prevents SSH timeouts from killing long jobs

### Example Commands Claude Uses
```bash
# Read a file
ssh -T timtam "cat ~/qss-pulsegen/src/main.py"

# Edit a file
ssh -T timtam "cat > ~/path/to/file << 'EOF'
file contents here
EOF"

# Run a script
ssh -T timtam "cd ~/qss-pulsegen && python test.py"
```

### ⚠️ Heredoc Quoting Gotcha

When writing Python files via SSH heredoc, **quotes inside f-strings get stripped** if not properly escaped.

**Problem:**
```bash
ssh -T server 'cat > file.py << '\''EOF'\''
print(f"Time: {time.strftime('%H:%M:%S')}")
EOF'
```
Results in: `time.strftime(%H:%M:%S)` — quotes lost!

**Solutions:**
1. **Use scp** to transfer files instead of heredoc for complex Python code
2. **Use escaped quotes**: `'\''%H:%M:%S'\''` (very ugly)
3. **Avoid f-strings with quotes** in heredoc-created files; use `.format()` or concatenation
4. **Write locally first**, then scp to remote

---

## Plot Styling Guidelines

All plots should have **bold black borders** around the plot area.

### Matplotlib
```python
for spine in ax.spines.values():
    spine.set_linewidth(2)
    spine.set_color("black")
```

### Plotly
```python
fig.update_layout(
    xaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
    yaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
)
```
