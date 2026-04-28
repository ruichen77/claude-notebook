# TWPA Noise Measurement Project

Traveling Wave Parametric Amplifier noise characterization and analysis.

## ⚠️ Measurement Pre-Launch Checklist — ALWAYS FOLLOW

**Before launching ANY `record_trace.py` campaign > 30 min on fridge36B, walk through the 7-item checklist in the auto-memory:** `feedback_measurement_pre_launch_checklist.md`.

The three recurring traps that have cost ~12 hr of cryostat time across sessions:

1. **Switch routing** — wrapper inherits stale `set_switches AB1` from a copied template instead of the device's actual relay (e.g. D3 needs `AB4`). Result: signal goes to the wrong DUT, looks like everything's working. Always read `relay: [N]` from `benchmark/exp_params.yaml` per device.
2. **MC-PUMP routing** — wrapper sets the chain switches but forgets `set_switches PUMP04_to_P4` (or per-device variant). PUMP04 is on but routes to whatever the previous campaign left the mux at. Always set both signal AND pump-mux per device.
3. **Thru subtraction missing in build** — `screening_thru_cal_*.csa` cal plane is upstream of the device. Raw S21 reads ~2× the true device gain (residual ~18 dB HEMT chain). `gain_map/fine_process.py` subtracts thru post-hoc; new build scripts that read raw CSVs forget this. Sanity check: heatmap peak should be 0–25 dB, NEVER ≥ 30 dB on a healthy TWPA.

**The 30-second smoke test at peak op-point (compare device gain vs gain_map ±3 dB after thru-subtract) catches all three.** Don't skip it. Global Rule A applies to measurements as well as sims.



## Code Repos (on servers)

- **Noise measurement**: `timtam:/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/`
  - Run: `python -m noise_measurement run -c config.yaml`
  - Resume crashed run: `python -m noise_measurement run -c config.yaml --resume noise_test1_DEVICE/`
  - Full docs: `~/.claude/docs/twpa_noise/noise_measurement_package.md`
- **Noise analysis**: local `~/repos/noise_analysis_v2/`

## Measurement Locations

- **fridge36B** (timtam): `/nas-data1/systems/fridge36B/`
  - Setup guide: `~/.claude/docs/measurement_folder_setup.md`
  - `~/.localrc` defines `current` alias → active measurement folder
- **fridge1F Big Endeavour** (narya): `/nas-data1/systems/fridge1F/20250728_E_BE/BE001`

## Key Docs (in ~/.claude/docs/)

- `measurement_folder_setup.md` - how to create new cooldown folders
- `twpa_noise/noise_analysis_v2_readme.md` - noise analysis package overview
- `twpa_noise/custom_list_mode.md` - custom sweep configurations
- `twpa_noise/custom_list_visualization_plan.md` - visualization pipeline
- `twpa_noise/equal_gain_finder.md` - finding equal-gain pump configs
- `twpa_noise/multi_device_enhancement.md` - multi-device overlay
- `twpa_noise/noise_analysis_v2_custom_list.md` - custom list implementation
- `twpa_noise/twpa_hb_simulator.md` - harmonic balance simulator
- `twpa_noise/cooldown_20260223_RANNCTHRU_RANV505HS.md` - current cooldown wiring & device table
- `twpa_noise/cooldown_20260223_plan.md` - current cooldown measurement plan & status
- `twpa_noise/noise_measurement_package.md` - noise measurement package docs (CLI, config, resume feature)

## Noise Analysis Rules

**ALWAYS follow these when processing noise temperature data:**

1. **Identify device type first** — is it a TWPA or a thru reference? Ask for clarification if unclear.
   - TWPA: active parametric amplifier (has pump, produces gain)
   - Thru: passive reference (no pump, used for calibration)
2. **Always use `idler_gain: 1` for TWPA devices** — TWPAs (both 3WM and 4WM) have idler noise contribution. Setting `idler_gain: 0` ignores this and inflates QE.
   - `idler_gain: 0` is ONLY correct for thru references
3. **Filter unphysical QE** — exclude fits with QE > 1.0 or QE <= 0 (unphysical, usually near parametric oscillation threshold)
4. **QE formula** — v2 package uses `QE = 1/(0.5 + T_exc/(hf/k))` where `T_exc` is excess noise temperature from Y-factor fit (same as legacy). Verified in `noise_calculations.py:134`.
5. **QE vs Gain plots** — sort traces by pump power (not gain) to avoid zigzag artifacts from overdriven points
6. **SQL line at QE = 1.0** — with `QE = 1/(0.5 + T_exc/(hf/k))`, a SQL amplifier adds half a photon (`T_exc = hf/(2k)`, `A = 0.5`) giving `QE = 1/(0.5+0.5) = 1.0`. Do NOT use 0.5.
7. **Always use noise_analysis_v2 package** for metrics extraction — it handles thru reference matching and Y-factor fitting correctly. Do not write custom extraction scripts.
8. **Resume mode ignores config timing** — `--resume` loads switch_wait_time from saved experiment config. If switch settling is needed after a switch change, restart from scratch instead of resuming.

## VNA Scan / Gain Map Rules

**ALWAYS follow these when processing 2D gain map data from vnascan:**

1. **Raw S21 data is already thru-calibrated** — the `record_trace` script subtracts the nestedthru reference during acquisition. Do NOT subtract any additional reference (unpumped device S21, nestedthru, etc.) when computing gain from 2D sweep data. The raw S21 values ARE the thru-calibrated gain.
2. **Gain = raw S21 value** — just read the S21 magnitude directly from the CSV and average over the probe band. No background subtraction needed.

## JTWPA Simulation Campaign Numbering Convention

**All simulation runs MUST be numbered using the Phase system: Campaign N, Phase NA/NB/NC/...**

This tracks the thinking trajectory — each campaign is a high-level investigation goal, and each phase within it is a specific experiment or parameter sweep. Sequential phases within a campaign show how one result led to the next.

**Format**: `Phase <Campaign><Letter>` — e.g., Phase 5A, Phase 5B, Phase 6A, Phase 7A

**Rules**:
- **New campaign** = new scientific question or investigation direction (e.g., "Campaign 5: Cc coupling sweep", "Campaign 6: No-RPM spur investigation", "Campaign 8: Spur origin diagnostics")
- **New phase** = new simulation run or parameter sweep within that campaign, lettered sequentially (A, B, C, D, E, F...)
- **Phase naming** should reflect what changed: e.g., "Phase 5E: 2D Cc × Ppump sweep", "Phase 6B: Pump frequency sweep"
- **Always document** in `catalogue/YYYY-WNN.md` with the phase ID prominently in the heading
- **Cross-reference** between phases to show the logical chain (e.g., "Phase 6B follows from Phase 6A finding that spurs are MI, now sweeping fpump to track spur trajectory")

**Current campaigns**:
- Campaign 1–3: Early exploration (200–1956 cell, Lbond, noise methods)
- Campaign 4: Chirped stub investigation (alpha, stepped, random)
- Campaign 5: Cc coupling capacitance sweep (5A–5F: coarse → fine → 2D)
- Campaign 6: No-RPM bare JJ line (6A: power sweep, 6B: fpump sweep)
- Campaign 7: Gaussian pulse + HB comparison (7A: HB RPM, 7B: HB no-RPM fpump)
- Campaign 8: JoSIM spur origin diagnostics (8.1–8.5: Nj, Ppump, R, Cj, Cg sweeps)

**Circuit loss model note**: Phase 5E, 6A, 6B all used **lossless capacitors** (tan_delta=0). Multitone CW and HB lossy runs used explicit dielectric loss (Q_Cg=360, Q_Cj=300). Always document the loss model in each phase entry.

## Presentation Folder Convention

**Location**: `~/Library/CloudStorage/Box-Box/Presentation/TWPA_spur_timedomain/`

**Structure**: Each topic gets its own subfolder containing the PPTX deck AND its associated interactive HTML plots.

```
TWPA_spur_timedomain/
├── Campaign5A_Cc_scaling/       # deck + plots for Campaign 5A
├── Phase5EF_Cc_Ppump_merged/    # deck + plots for Phase 5E+5F
├── HB_vs_JoSIM_gain_comparison/ # deck + all comparison HTMLs
├── Spur_Diagnostics/            # deck + sweep 1-5 spectra/scatter HTMLs
└── ...
```

**Rules**:
- **When saving a new deck**: create a folder named after the topic. Put the `.pptx` and all associated `.html` plots in the same folder.
- **When the topic matches an existing folder**: reuse it — add new plots/decks to the existing folder.
- **New topic** = new folder. Judge by whether the scientific question is different.
- **Never leave loose files** in the root directory — everything goes in a topic folder.
- **HTML plot naming**: use short descriptive names (e.g., `rpm_noise_s21.html`, `sweep1_Nj_spectra.html`), not long generated names.

## Server Access

timtam and narya require 2FA (SSH ControlMaster — see global CLAUDE.md).
