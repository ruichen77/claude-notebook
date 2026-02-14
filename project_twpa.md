# TWPA Noise Measurement Project

Traveling Wave Parametric Amplifier noise characterization and analysis.

## Code Repos (on servers)

- **Noise measurement**: `timtam:/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/`
  - Run: `python -m noise_measurement run -c config.yaml`
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

## Server Access

- **timtam**: 2FA required (SSH ControlMaster - user must SSH first)
- **narya**: 2FA required (SSH ControlMaster - user must SSH first)
