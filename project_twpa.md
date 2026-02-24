# TWPA Noise Measurement Project

Traveling Wave Parametric Amplifier noise characterization and analysis.

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
4. **QE formula difference** — legacy and v2 packages use different formulas:
   - Legacy: `QE = 1/(0.5 + T_sys/(hf/k))`
   - v2: `QE = hf/(k * T_sys)`
   - These diverge at low T_sys. Always use the same formula when comparing across devices.
5. **QE vs Gain plots** — sort traces by pump power (not gain) to avoid zigzag artifacts from overdriven points

## VNA Scan / Gain Map Rules

**ALWAYS follow these when processing 2D gain map data from vnascan:**

1. **Raw S21 data is already thru-calibrated** — the `record_trace` script subtracts the nestedthru reference during acquisition. Do NOT subtract any additional reference (unpumped device S21, nestedthru, etc.) when computing gain from 2D sweep data. The raw S21 values ARE the thru-calibrated gain.
2. **Gain = raw S21 value** — just read the S21 magnitude directly from the CSV and average over the probe band. No background subtraction needed.

## Server Access

- **timtam**: 2FA required (SSH ControlMaster - user must SSH first)
- **narya**: 2FA required (SSH ControlMaster - user must SSH first)
