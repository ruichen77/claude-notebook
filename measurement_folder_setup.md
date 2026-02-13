# Measurement Folder Setup Guide for fridge36B

This document describes how to initialize a new measurement (cooldown) folder on timtam at `/nas-data1/systems/fridge36B/`.

## Folder Naming Convention

```
YYYYMMDD_description
```
- **Date**: 8 digits (e.g., `20260202`)
- **Separator**: underscore
- **Description**: Device names or experiment description (e.g., `RANv401_RANv402`)

## Folder Structure

```
/nas-data1/systems/fridge36B/YYYYMMDD_description/
├── switch_map.yaml              # Instrument addresses and switch mappings
└── benchmark/
    ├── batch_job.sh             # Main batch job script
    ├── benchmark_qla.yaml       # QLA benchmark config
    ├── cascade_tuning.sh        # Cascade TWPA tuning script
    ├── config_cascaded_twpa.yaml
    ├── config_single_twpa.yaml
    ├── config_thru.yaml
    ├── exp_params.yaml          # Experiment parameters and device definitions
    ├── heater_off.sh            # Heater control script
    ├── ibmqc.yaml
    ├── instruments.yaml         # Instrument definitions
    ├── noise_analysis.yaml      # Noise measurement config
    ├── noise_test.sh            # Noise test script
    └── vna_scan/                # VNA S21 measurements
        ├── 2d_sweep.sh          # 2D parameter sweep (freq vs power)
        ├── batch_job2.sh
        ├── batch_job3.sh
        ├── batch_job4.sh
        ├── batch_job_unpumped.sh  # Unpumped S21 collection
        ├── exp_params.yaml      # Device definitions for VNA scans
        └── record_trace.yaml    # Trace recording config
```

## Setup Steps

### 1. Create the folder structure

```bash
# Create main folder
mkdir -p /nas-data1/systems/fridge36B/YYYYMMDD_description

# Create benchmark and vna_scan subfolders
mkdir -p /nas-data1/systems/fridge36B/YYYYMMDD_description/benchmark/vna_scan
```

### 2. Copy files from the most recent cooldown folder

Find the most recent folder:
```bash
ls -la /nas-data1/systems/fridge36B | tail -20
```

Copy the following files from the previous cooldown:

**Root level:**
- `switch_map.yaml`

**benchmark/ folder:**
- `batch_job.sh`
- `benchmark_qla.yaml`
- `cascade_tuning.sh`
- `config_cascaded_twpa.yaml`
- `config_single_twpa.yaml`
- `config_thru.yaml`
- `exp_params.yaml`
- `heater_off.sh`
- `ibmqc.yaml`
- `instruments.yaml`
- `noise_analysis.yaml`
- `noise_test.sh`

**benchmark/vna_scan/ folder:**
- `2d_sweep.sh`
- `batch_job2.sh`
- `batch_job3.sh`
- `batch_job4.sh`
- `batch_job_unpumped.sh`
- `exp_params.yaml`
- `record_trace.yaml`

### 3. Update the `current` alias

Edit `~/.localrc` and update the `current` alias:
```bash
alias current="cd /nas-data1/systems/fridge36B/YYYYMMDD*/benchmark"
```

Then reload: `source ~/.localrc`

## Files to Modify for New Devices

After copying, update these files with new device names and switch positions:

1. **`switch_map.yaml`** - Update switch mappings if hardware changed
2. **`benchmark/exp_params.yaml`** - Update device definitions
3. **`benchmark/vna_scan/exp_params.yaml`** - Update device definitions for VNA scans
4. **`benchmark/vna_scan/batch_job_unpumped.sh`** - Update device names and switch positions

## VNA Scan Folder Purpose

The `vna_scan/` folder is used for:
1. **Unpumped S21 traces** - Baseline measurements of thru standards and DUTs (no pump)
2. **Single TWPA pump sweeps** - Finding optimal pump frequency and power for max gain in readout band (6.5-7.5 GHz)

### Workflow:
1. Run `batch_job_unpumped.sh` - Collect unpumped S21 for thru and all DUTs
2. Run `2d_sweep.sh` - Sweep pump freq vs power to find optimal operating point
3. Results stored in timestamped folders with data/, plot/, and metadata.yaml

---

## Noise Temperature Measurement

### Package Location

The noise measurement package is located at:
```
/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/
```

### Usage

```bash
cd /nas-data1/systems/fridge36B/YYYYMMDD_description/benchmark

# Run noise measurement with a config file
python -m noise_measurement run -c config_thru.yaml

# Validate config without running
python -m noise_measurement validate -c config_thru.yaml

# Estimate measurement duration
python -m noise_measurement estimate -c config_thru.yaml

# Check status of running/completed experiment
python -m noise_measurement status /path/to/noise_test_folder

# Generate example config
python -m noise_measurement generate-config -t thru -o config_example.yaml
```

### Config Files

Three config templates are provided in each benchmark folder:

| Config File | Device Type | Description |
|-------------|-------------|-------------|
| `config_thru.yaml` | thru | Reference measurement (no TWPA, just thru standard) |
| `config_single_twpa.yaml` | single_twpa | Single-stage TWPA measurement |
| `config_cascaded_twpa.yaml` | cascaded_twpa | Cascaded (two-stage) TWPA measurement |

### Config File Structure

```yaml
general:
  base_path: /nas-data1/systems/fridge36B/YYYYMMDD_description/benchmark  # UPDATE THIS!
  library_path: /nas-data0/systems/fridge36B/Ruichen_library
  exp_params_file: exp_params.yaml
  timing:
    switch_wait_time: 1800        # 30 min wait after switch change
    temperature_stabilization: 1200  # 20 min wait for temp stability

measurement:
  name: nestedthru_reference      # Output folder will be noise_test1_{name}
  device: nestedthru
  device_type: thru               # thru, single_twpa, or cascaded_twpa
  switch_position: AB6            # Switch position for this device

  probe:
    instrument: PUMP_PROBE
    frequencies: "np.linspace(6.5e9, 7.5e9, 21)"
    power: 0

  spectrum_analyzer:
    instrument: sa
    receiver: BREC
    channel: 1
    points: 2001
    freq_span: 10000
    bw: 91
    video_bw: 5.1
    average_count: 10
    trigger_source: EXT

sweep:
  loop_order:
    - temperature
  axes:
    temperature:
      - 0.2
      - 0.3
      - 0.4
      - 0.5
      # ... add more temperatures as needed

live_plot:
  enabled: true
  refresh_interval: 30
```

### Workflow for Noise Measurement Campaign

1. **First: Run thru reference measurement**
   - Update `config_thru.yaml` with correct `base_path` for current cooldown
   - Run: `python -m noise_measurement run -c config_thru.yaml`
   - This generates the reference data needed for all subsequent DUT measurements

2. **Then: Run DUT measurements**
   - Update `config_single_twpa.yaml` or `config_cascaded_twpa.yaml`
   - Set correct device name, switch position, pump settings
   - Run: `python -m noise_measurement run -c config_single_twpa.yaml`

### Important Notes

- **Always run thru reference first** - DUT measurements need this as baseline
- **Update `base_path`** in config to point to current cooldown folder
- **Check switch_position** matches the device in switch_map.yaml
- Measurements are saved to `{base_path}/noise_test1_{measurement.name}/`

---

*Last updated: 2026-02-02*
