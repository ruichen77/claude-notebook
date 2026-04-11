# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment Overview

This is a development environment focused on computational electromagnetics, quantum computing simulations, and CUDA-accelerated scientific computing. The system runs Ubuntu Linux with NVIDIA GPU support and uses conda for Python environment management.

## Key Projects

### Palace (repos/palace)
- 3D finite element solver for computational electromagnetics
- CMake-based C++ project using MFEM and libCEED libraries
- Supports eigenmode calculations, frequency/time domain simulations, and adaptive mesh refinement
- GPU acceleration via CUDA/HIP

**Build commands:**
```bash
cd repos/palace
mkdir -p build && cd build
cmake .. [OPTIONS]
make -j$(nproc)
```

**Documentation:** Run `julia make.jl` from `docs/` directory

### cuOPO (repos/cuOPO)
- CUDA-based optical parametric oscillator simulator
- Implements Split-Step Fourier Method for solving coupled-wave equations
- Uses CUFFT library for GPU-accelerated FFT

**Compile and run:**
```bash
cd repos/cuOPO/src
chmod +x cuOPO.sh
./cuOPO.sh
```

**Compilation pattern:**
```bash
nvcc cuOPO.cu -D<REGIME> -D<CRYSTAL> [-DTHREE_EQS] --gpu-architecture=sm_XX -lcufftw -lcufft -o cuOPO
```
- `<REGIME>`: `CW_OPO` or `NS_OPO`
- `<CRYSTAL>`: MgO:PPLN or MgO:sPPLT
- `sm_XX`: GPU architecture (sm_60 for Pascal, sm_75 for Turing)

### CUDA Projects (repos/cuda_projects)
- Basic CUDA examples including vector addition and matrix multiplication
- Used for learning and testing CUDA programming concepts

## System Configuration

**Conda environments:**
- `base`: Default environment with miniconda3
- `ocr`: OCR processing environment
- `qubits`: Quantum computing simulations
- `tensor`: Tensor operations and deep learning

**Activate environment:**
```bash
conda activate <env_name>
```

**GPU Information:**
- CUDA toolkit installed (check version with `nvcc --version`)
- Typical GPU architectures: Pascal (sm_60), Turing (sm_75)

## Development Tools

**Spack Package Manager:**
- Located at `~/spack/`
- Used for installing scientific computing packages
- Palace can be installed via: `spack install palace`

**Common Commands:**
```bash
# Check GPU status
nvidia-smi

# CUDA compilation
nvcc -o output_file source.cu --gpu-architecture=sm_XX

# CMake project build
cmake -B build -S .
cmake --build build -j$(nproc)
```

### Local LLM (Ollama + llama-batch)
- Llama 3.1 8B Q4 running locally via Ollama on RTX 3060 Ti
- Ollama API at `http://localhost:11434`
- Use `llama-batch` CLI tool for bulk/batch processing tasks

**llama-batch usage:**
```bash
# Single prompt
echo '{"prompt": "Summarize: ..."}' | llama-batch

# Batch of prompts with system prompt
echo '[
  {"id": "1", "system": "You are a classifier.", "prompt": "Classify: ..."},
  {"id": "2", "system": "You are a classifier.", "prompt": "Classify: ..."}
]' | llama-batch --concurrency 4 --temperature 0.2
```

**Options:**
- `--model`: Model name (default: `llama3.1:8b-instruct-q4_K_M`)
- `--concurrency`: Parallel requests (default: 2, recommend 2-4 for GPU)
- `--temperature`: Sampling temperature (default: 0.1)

**Best for:** summarization, classification, tagging, extraction, translation, simple Q&A, reformatting — any simple but high-volume task that can be offloaded from Claude API.

**Output:** JSON with `total_tasks`, `completed`, `failed`, `elapsed_seconds`, and `results` array.

## Architecture Notes

**Palace Project Structure:**
- Uses superbuild pattern with external dependencies in `extern/`
- Configuration files in JSON format located in `examples/`
- Supports multiple mesh formats and parallel execution via MPI
- Testing framework under `test/unit/`

**CUDA Development:**
- Check GPU architecture before compiling (use `nvidia-smi` or CUDA samples)
- CUFFT library required for Fourier transforms on GPU
- Compilation requires proper sm_XX flag matching GPU architecture

## Working with Projects

**When modifying Palace:**
- C++17 required, ensure compiler compatibility
- CMake version 3.21 or later required
- Run tests from build directory: `ctest`
- Documentation uses Julia-based build system

**When modifying cuOPO:**
- Preprocessor flags control simulation regime and crystal type
- Output files are .dat format with separate real/imaginary parts
- Verify GPU architecture flag matches hardware before compilation

**General Guidelines:**
- Most scientific computing projects use conda environments - activate appropriate environment before development
- GPU code requires CUDA drivers and toolkit properly installed
- Many projects use CMake - prefer out-of-source builds in `build/` directories


## Circuit Diagram Quality Rules

When producing any circuit schematic (CircuiTikZ, TikZ, matplotlib, sketch):
read `.claude/circuit_diagram_style.md` first. Canonical generator:
`crucible/src/crucible/viz/circuit.py`. Ten rules cover: drawing-must-match-netlist,
explicit lead wires on every node, leads into/out of block elements, representative
period for repeating chains, vertical stacking to avoid label collisions,
breathing room, page sizing, footnotes for non-obvious conventions, font
hierarchy, and always re-rendering to inspect visually.


## Domain Reasoning: Qubit/Coupler/Readout

Full prompt at: `~/openclaw-skills/quantum-reasoning/QUBIT_COUPLER_READOUT.md`
Key rules: never truncate transmon to 2 levels without justification; check dispersive limits before using chi formula; verify Purcell protection; check frequency collisions first for multi-qubit failures.


## Domain Reasoning: TWPA

Full prompt at: `~/openclaw-skills/quantum-reasoning/TWPA.md`
Key rules: gain ripple is almost never intrinsic to JJTL (check Fabry-Perot first); never confuse gain with quantum efficiency; use match_papers_array RPC (NOT match_papers); harmonic balance for steady-state, time-domain for transients; 2D pump sweep is the most diagnostic measurement.


## Notion Knowledge Base Index

Query `notion_pages` in Supabase before searching Notion directly.


### Analysis
- Bifurcation Analysis — HD3C Spur Onset Characterization -- [PARKED — not pursuing further. Coordinate transform resolve
- Campaign 14 Plan — JoSIM-as-SA: HD3C Spur Replication & Spatiotemporal Study -- Detailed execution plan for Campaign 14: replicate HD3C expe
- Coordinate Transform Resolves Bifurcation Exponent — Spur Onset Is Supercritical Parametric Oscillation -- [PARKED — resolved. β≈1.37 was coordinate artifact; true exp
- Experiment Proposal — MI Gain Profile Direct Measurement -- Proposed experiment to directly measure the MI gain profile 
- Experiment Proposal — Pump Depletion by MI (Energy Accounting) -- Proposed experiment to quantify pump power extracted by MI s
- Extended Spur-Threshold Ideas — Beyond the Original Five -- Extended ideas for spur suppression beyond original 5 approa
- Harbord Diplexer Port Mapping & Absorptive BPF Derivation -- Corrected port mapping for Harbord DPX S4P. Port 1=signal, 2
- Implementation Spec — IFFT Quantum Noise Source for SPICE Injection -- [SUPERSEDED — QNoise injection found unnecessary for spur re
- Implementation Spec — Multi-Tone Gain Measurement & Spur-Threshold-Gain Metric -- Spec for 21-tone CW gain measurement and spur-threshold-gain
- JTWPA Simulation Campaign — W12 Summary & Business Impact -- ~55 catalogue entries, ~500+ JoSIM/HB runs across 4 servers.
- JTWPA Spur Mechanism — Complete Picture (Campaign 14 Findings) -- Complete spur mechanism: parametric oscillation in Fabry-Per
- JTWPA Spur Simulation with Wirebond Reflections — JoSIM Campaign Results (200 to 1956 cells) -- JoSIM spur simulation at realistic cell count. Spurs emerge 
- Phase Matching & Cavity-Mode Spur Prediction Model — HD3C Analysis -- Analytical model for predicting discrete JTWPA spur frequenc
- Research North Star — TWPA Gain Limit & MI Suppression -- Primary research question: what limits TWPA gain? Current an
- Spur Bidirectionality — Traveling-Wave MI vs Fabry-Perot Resolution -- Investigation of whether JTWPA MI spurs require bidirectiona
- Spur Frequency Prediction — Challenges & Current Status (2026-03-30) -- [MERGED into "Phase Matching & Cavity-Mode Spur Prediction M
- Spur Origin Diagnostics — Campaign 8 Results (Sweeps 1–3) -- [SUPERSEDED by Campaign 14 — see "JTWPA Spur Mechanism — Com
- TWPA Chaos Detection via WRspice — Simulation Strategy & Literature Analysis -- Analysis of Guarcello 2024 chaos paper (DARTWARS). Simulatio
- TWPA Noise Characterization -- Device wiring and noise temperature characterization data fr
- TWPA Spur Mechanism Analysis — Modulation Instability & Mitigation Strategies -- [SUPERSEDED by Campaign 14 — see "JTWPA Spur Mechanism — Com
- Why VNA-Grade S21 from JoSIM Is Impractical — Lessons Learned -- [DECISION RECORD — VNA-grade S21 from JoSIM abandoned] VNA-g
- Wideband S21 Extraction from JoSIM — Analysis & Plan -- [ABANDONED PATH — kept to avoid retreading] Why Gaussian pul

### Diary
- Diary — 2026-03-17 — MI Spur Confirmation & Gaussian Pulse Method -- First JoSIM confirmation of MI as the spur generation mechan
- Diary — 2026-03-18 — HB RPM Sweep & Campaign 7 Scripts -- Harmonic balance RPM frequency sweep results for Campaign 7.
- Diary — 2026-03-18 — Spur Diagnostics & Loss Model Discovery -- Spur diagnostics sweeps. Discovery: Debye loss model causes 
- Diary — 2026-03-20 — Gain Discrepancy Root Causes & Ic Monte Carlo -- Resolved HB-vs-JoSIM gain gap: ip/2 convention (+6dB), Debye
- Diary — 2026-03-21 — Best-Match 16.3 dB & MI Spatial Profiles -- Compiled overnight results: Ic Monte Carlo, Debye models, Cc
- Diary — 2026-03-21 — Cosim Framework & DPX Analysis -- Implemented wave-domain co-simulation package (twpa-cosim). 
- Diary — 2026-03-22 — Embedded DPX & Cascaded TWPA -- Building DPX-TWPA1-DPX-TWPA2-BPF netlist. Embedded DPX simul
- Diary — 2026-03-22 — Spur = Superfluorescence Discovery -- Diary entry documenting the insight that MI spur emission an
- Diary — 2026-03-24 — Cosim Blockers & Pivot to TWPA-Only -- All cosim approaches hit fundamental blockers (Y-param DC si
- Diary — 2026-03-25 — MI Gain Race, Spur Classification & Phase 10D -- JoSIM V3 gain profile. MI gain race analysis. Spur classific
- Diary — 2026-03-25 — Signal Injection Bug & PWL Decimation Fix -- Fixed signal injection measurement artifact (different Vin l
- Diary — 2026-03-26 — No Activity -- No research activity recorded on 2026-03-26. Minor note on d
- Diary — 2026-03-26 — Overnight TWA SPDC Completions & Stalled V3 Sweeps -- Distributed TWA vacuum noise does NOT change MI behavior. Ov
- Diary — 2026-03-27 — Lock-in Validation & Wideband S21 Launch -- Lock-in extraction validated (matches FFT within 0.5 dB). Pa
- Diary — 2026-03-28 — R0 Spur Control & Diagonal Sweet Spot -- Proved R0 is sole RCSJ spur control (RN zero effect). Diagon
- Diary — 2026-03-30 — TWPA Spur Prediction Model Development -- Major session building spur prediction model. ABCD cavity mo
- Diary — 2026-04-05 — No Activity -- No research activity recorded on 2026-04-05. Placeholder dia
- Diary — Why Input-Only Noise Already Produces Spurs & TWA Implementation Plan -- Analysis and diary entry explaining why deterministic spurs 

### Literature
- Literature Survey — Phase Matching & MI Spur Frequency Prediction in JTWPAs -- Comprehensive survey of analytical (ABCD+CME, NLSE MI formul
- Literature Survey — TWPA S21 Extraction Methods in Time-Domain Simulation -- Surveyed 15+ JTWPA papers (all deterministic, no noise injec

### Procedure
- Claude Code CLAUDE.md Setup — New Machine -- Step-by-step procedure to set up the CLAUDE.md directory str
- Context Engine — Implementation Plan (Supabase + Notion + Obsidian) -- Implementation plan for automatic context injection into Cla
- Context Engine Setup — IBM Laptop -- Step-by-step setup for context engine on IBM laptop. Clone f
- Cooldown Folder Setup — fridge36B (timtam) -- Procedure to set up a new cooldown measurement folder on tim
- Cooldown Folder Setup (fridge36B) -- [DUPLICATE — see "Measurement Folder Setup — fridge36B"] Pro

### Reference
- JoSIM SA Frequency Resolution — What Controls It and How to Improve -- df = 1/T where T = tstop - t_skip. Default 2.5 MHz (12× coar
- JTWPA JoSIM Simulation Pipeline — Workflow Reference -- Complete JoSIM simulation workflow: netlist generation, runn
- Marika — Home Server (Pop!_OS) -- Home server specs (32 vCPU, 124 GiB RAM, RTX 3060 Ti). Used 
- Physics Context: Superconducting Qubit & DTC Simulation Reference -- Reference covering superconducting qubit physics relevant to
- Quantum Engineering Knowledge Base -- Top-level Notion workspace for quantum engineering research.
- TWPA Harmonic Balance Simulator -- Julia harmonic balance simulator using JosephsonCircuits.jl 
- TWPA Noise Analysis & Tools Reference -- Reference index for TWPA noise characterization tooling: noi
- TWPA Noise Measurement Package -- Automated noise temperature measurement system for TWPA devi
- TWPA Simulation Diary -- Parent folder indexing all TWPA simulation diary entries fro

### Tools
- llama-batch: Local LLM Batch Processor -- CLI tool at ~/.local/bin/llama-batch for offloading simple h
- OpenClaw: Server Operations Assistant Setup (marika) -- Operational guide for OpenClaw 24/7 server assistant on mari


## Obsidian Vault

Local vault at `/home/ruichenzhao/obsidian-vault`. Read files directly with Read tool.

- `research/` (128 notes)


## Context Engine

A SessionStart hook automatically injects active research state from Supabase at session start.
This includes: open investigations, running simulations, recent diary findings, key results, and parking lot items.

**Do NOT** manually query for this information at session start -- it is already injected.

To refresh context mid-session: query Supabase tables directly (investigation_nodes, simulation_runs, diary_entries, parking_lot).

Supabase project: `fqdmbvdaxjslgoqgclip`
Literature search: use `match_papers_array` RPC (NOT `match_papers`)


## Reference Library Protocol

**Before making technical claims** about microwave engineering, qubit physics, circuit QED, noise theory, or superconductivity, search the reference library:

```sql
SELECT * FROM search_all_references('your topic here', 5);
-- Searches across 25 textbooks (Pozar, Steer, Clerk, Girvin, Blais, etc.)
-- and 2000+ arXiv papers with full text
```

For book-specific lookup: `SELECT * FROM search_book_chunks('topic', 5, 'pozar_microwave_4ed');`
For design rules: `SELECT * FROM design_rules WHERE domain = 'twpa' OR rule ILIKE '%keyword%';`

**Cite sources** when using results: (Pozar §2.4 p.63, Clerk §V.B, arXiv:2003.00024).
**Flag contradictions** — if your answer contradicts a reference, say so explicitly.

### Challenge Protocol (use when reaching conclusions)

When you or the user reach a technical conclusion, **actively search for contradicting evidence** before accepting it:

1. Search for the opposite claim: `SELECT * FROM search_all_references('NOT <your_conclusion> OR alternative', 5);`
2. Search for boundary conditions where it breaks: `SELECT * FROM search_all_references('<your_conclusion> limit failure breakdown', 5);`
3. Check if any belief contradicts: `SELECT * FROM beliefs WHERE statement ILIKE '%keyword%' AND status = 'active';`
4. If contradiction found → present both sides and assess which evidence is stronger.

### Cross-Domain Innovation Patterns

When stuck on a problem, **force cross-domain search** to find transferable solutions:

- **Method transfer**: Search how a different subfield solves a structurally similar problem
  `SELECT * FROM search_all_references('<technique from another domain>', 5);`
- **Analogy bridge**: Map concepts between domains (e.g., "Purcell filter" ↔ "reflectionless filter for TWPA")
- **Constraint inversion**: If a parameter is treated as fixed, search for papers that tune it

Key cross-domain bridges for current work:
- Microwave filter design (Pozar Ch.8, Steer) ↔ TWPA spur suppression
- TLS physics (Martinis, Klimov) ↔ TWPA junction loss mechanisms
- Fluxonium disjoint support ↔ TWPA mode isolation
- Parametric amplifier theory (Clerk) ↔ TWPA gain/noise optimization
- Tunable coupler ZZ suppression ↔ TWPA inter-mode coupling


---
*CLAUDE.md auto-generated: 2026-04-09 04:47*
