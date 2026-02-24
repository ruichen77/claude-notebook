# Noise Measurement Package (`twpa_noise_measurement`)

Automated noise temperature measurement system for TWPA device characterization.

- **Version**: 2.0.0
- **timtam**: `/nas-data0/systems/fridge36B/Ruichen_library/twpa_noise_measurement/`
- **Local**: `~/repos/twpa_noise_measurement/`
- **GitHub**: `git@github.ibm.com:Ruichen-Zhao/twpa_noise_measurement.git` (branch: `dev`)
- **CLI**: `python -m noise_measurement run -c config.yaml`

---

## Package Structure

```
noise_measurement/
  __init__.py       - Package exports, version="2.0.0"
  __main__.py       - Entry point for `python -m noise_measurement`
  cli.py            - Click CLI: run, validate, estimate, generate-config, status
  core.py           - NoiseTemperatureMeasurement orchestrator class
  config.py         - YAML config loading, validation, numpy expression parsing
  sweep.py          - ParameterSweep (Cartesian), CustomListSweep, SweepOptimizer
  storage.py        - ExperimentStorage (Parquet index + CSV spectra), ThruReferenceLoader
  controllers.py    - TemperatureController, PumpController, SwitchController, MeasurementParameterManager
  acquisition.py    - SpectrumAnalyzerAcquisition, GainCalculator, snr_finder_probe()
  live_plot.py      - LiveDashboard (two HTML dashboards: spectra + metrics)
```

---

## CLI Commands

```bash
# Run measurement
python -m noise_measurement run -c config.yaml
python -m noise_measurement run -c config.yaml --dry-run
python -m noise_measurement run -c config.yaml --verbose

# Resume crashed measurement
python -m noise_measurement run -c config.yaml --resume noise_test1_DEVICE/

# Validate config without running
python -m noise_measurement validate -c config.yaml

# Estimate duration and show time breakdown
python -m noise_measurement estimate -c config.yaml

# Check experiment status (progress, SNR/gain ranges)
python -m noise_measurement status noise_test1_DEVICE/

# Generate example config
python -m noise_measurement generate-config -t single_twpa -o config.yaml
# Types: thru, single_twpa, cascaded_twpa
```

---

## Sweep Modes

### 1. Cartesian (`mode: cartesian`)
Full Cartesian product of all sweep axes. Loop order set by `loop_order` (first = outermost/slowest). Probe frequencies are always the innermost loop (fastest).

```yaml
sweep:
  mode: cartesian
  loop_order: [temperature, pump1_power]
  axes:
    temperature: [0.2, 0.3, 0.4, 0.5]
    pump1_power: "np.arange(-4, 0, 0.5)"
```

Valid sweep parameters: `temperature`, `pump1_power`, `pump1_frequency`, `pump1_phase`, `pump2_power`, `pump2_frequency`, `pump2_phase`, `probe_phase`.

### 2. Custom List (`mode: custom_list`)
User-specified pump configurations instead of Cartesian product. Fixed loop order: **temperature -> custom_config -> probe_freq** (innermost).

```yaml
sweep:
  mode: custom_list
  loop_order: [temperature]
  axes:
    temperature: [0.2, 0.3, 0.4, 0.5]
  custom_configs:
    - pump1_power: 13.0
      pump1_frequency: 8.00e9
      pump1_phase: 0
    - pump1_power: 14.0
      pump1_frequency: 8.06e9
      pump1_phase: 0
```

In custom_list mode, only `temperature` is allowed in `axes`. All pump parameters go in `custom_configs`.

---

## Data Storage

### Directory Layout
```
noise_testN_DEVICE/
  experiment_config.yaml      # Saved copy of input config
  experiment_index.parquet    # Master metadata (all measurements)
  data/
    00001.csv                 # Raw SA spectrum (freq_hz, magnitude_dbm)
    00002.csv
    ...
  plots/
    dashboard_spectra.html    # Live raw spectra (grouped by probe freq)
    dashboard_metrics.html    # Live metrics (SNR, gain, etc. vs probe freq)
  thru_reference/             # Copy of thru reference data (for non-thru devices)
```

### Write Strategy
- **Immediate**: Each measurement writes its CSV spectrum to `data/NNNNN.csv` right away
- **Buffered**: Index entries accumulate in memory, flush to Parquet every 10 measurements
- **Atomic writes**: Both index flush and HTML writes use temp-file-then-rename pattern
- **Auto-increment**: Directory name `noise_testN_DEVICE` auto-increments N if previous exists

### ExperimentStorage Key Methods
- `save_measurement(params, results, raw_spectrum)` -- saves one point
- `flush_index()` -- writes buffered entries to Parquet
- `get_index()` -- returns full DataFrame (flushes first)
- `get_spectrum(measurement_id)` -- loads CSV for one measurement
- `from_existing(experiment_dir)` -- class method for resume mode
- `close()` -- final flush

---

## Resume Feature

Recover from crashes without losing completed data.

### Usage
```bash
python -m noise_measurement run -c config.yaml --resume noise_test1_DEVICE/
```

### How It Works
1. Loads config from the crashed run's saved `experiment_config.yaml` (ignores the `-c` config file for settings)
2. Determines resume point via `_determine_resume_index()`:
   - Counts entries in Parquet index (`index_count`)
   - Counts CSV files in `data/` directory (`data_file_count`)
   - Uses `max(index_count, data_file_count)` as completed count (handles unflushed buffer)
3. Skips completed measurements in the main loop
4. Forces full hardware reconfiguration on first resumed point (`prev_point = None`)
5. Appends new data to the same experiment directory (continues file numbering)

### Key Code Paths
- `cli.py`: `--resume` option on `run` command
- `core.py`: `_determine_resume_index()`, resume logic in `run()` loop
- `storage.py`: `ExperimentStorage.from_existing()` class method

---

## Config Structure

```yaml
general:
  base_path: /path/to/benchmark           # Experiment output base directory
  exp_params_file: exp_params.yaml         # ibmqc instrument params file
  library_path: /path/to/library           # Path for heating script, etc.
  timing:
    switch_wait_time: 1800                 # Switch settling (seconds)
    temperature_stabilization: 1200        # Per temperature setpoint (seconds)
    base_temperature: 0.02                 # Return temp after measurement

measurement:
  name: DEVICE_noise                       # Used in folder name: noise_testN_<name>
  device: DEVICE-NAME                      # Device identifier
  device_type: single_twpa                 # thru | single_twpa | cascaded_twpa
  switch_position: AB2                     # RF switch position
  thru_reference: /path/to/thru            # Required for non-thru devices
  probe:
    instrument: PUMP_PROBE                 # Probe signal generator name
    frequencies: "np.linspace(6.5e9, 7.5e9, 21)"  # Numpy expression or list
    power: -15.0                           # dBm (default: -15)
    phase: 0.0                             # degrees (default: 0)
  spectrum_analyzer:
    instrument: sa                         # SA instrument name
    receiver: BREC                         # Receiver channel (default: BREC)
    channel: 1                             # SA channel (default: 1)
    points: 2001                           # Trace points (default: 2001)
    freq_span: 10000                       # Span in Hz (default: 10000)
    bw: 91                                 # RBW in Hz (default: 91)
    video_bw: 5.1                          # VBW in Hz (default: 5.1)
    average_count: 10                      # SA averages (default: 10)
    trigger_source: EXT                    # Trigger (default: EXT)
  pumps:
    pump1:
      instrument: PUMP01                   # Pump signal generator name
      frequency: 8.06e9                    # Hz
      power: 14.0                          # dBm
      phase: 0.0                           # degrees
      rf_out: 1                            # 1=ON, 0=OFF (default: 1)
    pump2:                                 # Only for cascaded_twpa
      instrument: PUMP04
      frequency: 8.39e9
      power: 10.0
      phase: 90.0
      rf_out: 1

sweep:
  mode: custom_list                        # cartesian | custom_list
  loop_order: [temperature]                # Outer-to-inner ordering
  axes:
    temperature: [0.2, 0.3, 0.4, 0.5]     # Numpy expression or list
  custom_configs:                          # Only for custom_list mode
    - pump1_power: 13.0
      pump1_frequency: 8.00e9
      pump1_phase: 0

live_plot:
  enabled: true                            # Enable live dashboards (default: true)
  refresh_interval: 300                    # HTML auto-refresh seconds (default: 30)
  raw_spectra_enabled: true                # Generate spectra dashboard (default: true)
```

### Config Parsing Notes
- Numpy expressions in YAML: `"np.linspace(6.5e9, 7.5e9, 21)"`, `"np.arange(-4, 0, 0.5)"`, etc.
- Security: only `np.arange`, `np.linspace`, `np.logspace`, `np.array`, `np.around`, and list literals are allowed
- SA numeric values auto-converted to proper types (YAML may parse `1e4` as string)

---

## Hardware Controllers

### TemperatureController
- Calls `lakeshore/heatingTempSweep_Pchip.py` script for temperature changes
- Caches current temperature to skip redundant changes
- Saves temperature CSV to experiment directory

### PumpController
- Direct instrument control via ibmqc instrument objects
- Caches pump state (frequency, power, phase) to skip redundant commands
- `all_rf_off()` calls the `all_rf_off` CLI command

### SwitchController
- Calls `set_switches` CLI command
- Waits `switch_wait_time` seconds after switching

### MeasurementParameterManager
- Wraps all three controllers
- Smart caching: only sends commands for parameters that actually changed
- `apply_parameters(params)` returns set of changed parameter names

---

## Acquisition Pipeline

Per measurement point:
1. `set_probe(freq, power, phase)` -- configure probe signal generator
2. `configure_sa_for_snr(center_freq)` -- set SA center, span, RBW, VBW, averaging
3. `acquire_spectrum()` -- wait for averaging, read SA data
4. `snr_finder_probe(freq, mag)` -- find peak (signal), median (background), compute SNR
5. `GainCalculator.calculate_metrics()` -- compare signal/SNR against thru reference

### GainCalculator
- Loads thru reference from `ThruReferenceLoader`
- Gain = device_signal_dBm - thru_signal_dBm
- SNR improvement = device_SNR_dB - thru_SNR_dB
- Matches by temperature and frequency with tolerance

---

## Live Dashboards

Two auto-refreshing HTML files (using Plotly.js):

1. **`dashboard_spectra.html`** -- Raw SA traces in a subplot grid (one subplot per probe frequency), colored by sweep parameters. Updated every 10 measurements.
2. **`dashboard_metrics.html`** -- Tabbed view of SNR, Signal, Background, Gain, SNR Improvement vs probe frequency. One trace per sweep parameter combination. Updated every measurement.

Both use `<meta http-equiv="refresh">` for auto-refresh and atomic file writes.

---

## Typical Workflow

```bash
# 1. Create config in benchmark directory
cd /nas-data1/systems/fridge36B/COOLDOWN/benchmark/
vi config_DEVICE.yaml

# 2. Validate config
python -m noise_measurement validate -c config_DEVICE.yaml

# 3. Estimate duration
python -m noise_measurement estimate -c config_DEVICE.yaml

# 4. Launch in tmux
tmux new -d -s noise 'cd /path/to/benchmark && python -u -m noise_measurement run -c config_DEVICE.yaml 2>&1 | tee measurement.log'

# 5. Monitor
tail -f measurement.log
# Or open plots/dashboard_metrics.html in browser

# 6. If crash, resume
python -m noise_measurement run -c config_DEVICE.yaml --resume noise_test1_DEVICE/

# 7. Analyze with noise_analysis_v2
noise-analysis-v2 run analysis_config.yaml
```

---

## Known Issues

- **live_plot tuple formatting warning**: Cosmetic warning when sweep has a single parameter, does not affect data
- **Git HTTPS on timtam**: Does not work non-interactively. Deploy code changes via SCP instead
- **Resume reads saved config**: `switch_wait_time` and other settings come from the saved `experiment_config.yaml`, not the current `-c` config file
- **Unreachable code in controllers.py**: `TemperatureController.set_temperature()` has duplicate success handling after the `finally` block (dead code, harmless)
