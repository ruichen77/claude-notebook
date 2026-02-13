# Noise Analysis v2

Noise analysis package for quantum amplifier characterization (TWPA, cascaded devices).

**Supports ONLY the new `experiment_index.csv` format.** For legacy format (metadata.yaml), use the existing `noise_analysis` package.

## Installation

```bash
# From source
cd noise_analysis_v2
pip install .

# Development mode
pip install -e .

# Or just add to PYTHONPATH
export PYTHONPATH="${PYTHONPATH}:/path/to/noise_analysis_v2"
```

## Quick Start

### CLI Usage

```bash
# Generate example config
noise-analysis-v2 init

# Run full analysis pipeline
noise-analysis-v2 run analysis_config.yaml

# Load and inspect data
noise-analysis-v2 load /path/to/noise_test1_device

# Compute metrics directly
noise-analysis-v2 compute /path/to/device --thru /path/to/thru -o metrics.csv

# Generate plots from metrics
noise-analysis-v2 plot metrics.csv -o ./plots
```

### Python API

```python
from noise_analysis_v2 import NoiseAnalyzer

# Full pipeline from config
analyzer = NoiseAnalyzer()
results = analyzer.run('analysis_config.yaml')
```

```python
from noise_analysis_v2 import (
    DataLoader, 
    compute_noise_metrics,
    ThruReferenceProcessor,
    generate_standard_plots
)

# Programmatic usage
loader = DataLoader()
df = loader.load('/path/to/noise_test1_device', label='My Device')

# Process thru reference
thru_proc = ThruReferenceProcessor()
thru_df = thru_proc.load_reference('/path/to/thru')
df = thru_proc.match_and_compute_gain(df, thru_df)

# Compute noise metrics
metrics = compute_noise_metrics(df, pump_freq_col='pump1_freq_hz')

# View results
print(metrics[['probe_freq_hz', 'quantum_efficiency', 'T_sys_K', 'fit_r2']])

# Generate plots
plots = generate_standard_plots(metrics, output_folder='./plots')
```

## Data Format

This package requires the **new format** with `experiment_index.csv`:

```
measurement_folder/
├── experiment_config.yaml      # Optional config
├── experiment_index.csv        # Required: all measurement metadata
└── data/
    ├── 00001.csv              # Individual trace files
    ├── 00002.csv
    └── ...
```

### Required Columns in experiment_index.csv

| Column | Description | Unit |
|--------|-------------|------|
| `temperature_k` | Bath temperature | K |
| `probe_freq_hz` | Probe/signal frequency | Hz |
| `pump1_power_dbm` | Pump 1 power | dBm |
| `pump1_freq_hz` | Pump 1 frequency | Hz |
| `signal_mag_dbm` | Peak signal magnitude | dBm |
| `background_mag_dbm` | Background (median) magnitude | dBm |
| `data_file` | Path to trace CSV | - |

Optional: `pump2_power_dbm`, `pump2_freq_hz`, `pump1_phase_deg`, `pump2_phase_deg`, `snr_db`, `gain_db`

## Configuration File

```yaml
# analysis_config.yaml

# Thru reference path aliases (optional)
thru_paths:
  my_thru: "/path/to/thru/measurement"

# Data sources
data_sources:
  - path: /path/to/noise_test1_device
    label: "Device Run 1"
    thru_reference: /path/to/thru
    
  - path: /path/to/noise_test2_device
    label: "Device Run 2"
    thru_path_ref: my_thru  # Use alias

# Analysis parameters
analysis:
  pump_freq_col: pump1_freq_hz  # Column for idler calculation
  idler_gain: 0                 # Conversion gain (0 for thru)
  r2_threshold: 0.5             # Minimum R² for valid fit

# Custom plots (omit for default set)
plots:
  - type: line
    x: probe_freq_hz
    y: quantum_efficiency
    group_by: [pump1_power_dbm]
    title: "QE vs Frequency"
    
  - type: heatmap
    x: pump1_power_dbm
    y: pump2_power_dbm
    z: quantum_efficiency
    slice_by: [probe_freq_hz]

# Output
output:
  folder: ./analysis_output
  formats: [html, png]
```

## Physics

### Y-Factor Method

Input noise temperature with quantum correction:
```
T_in = (hf/2k) × coth(hf / 2kT_bath)
```

For TWPA with idler contribution:
```
T_in_eff = T_in(f_signal) + G × T_in(f_idler)
where f_idler = 2×f_pump - f_signal
```

### Linear Fit

```
P_out = slope × T_in + intercept
```

### Derived Quantities

```
T_exc = -intercept / slope       # Excess noise temperature (x-intercept)
T_sys = T_exc + hf/2k            # System noise temperature
A = T_exc / (hf/k)               # Excess photon number
QE = 1 / (0.5 + A)               # Quantum efficiency
```

## Output Files

After running analysis:

```
analysis_output/
├── raw_data.csv          # Combined raw measurements
├── noise_metrics.csv     # Computed T_exc, T_sys, QE, fit stats
├── thru_reference.csv    # Thru data (if used)
└── plots/
    ├── line_quantum_efficiency_vs_probe_freq_hz.html
    ├── line_quantum_efficiency_vs_probe_freq_hz.png
    ├── heatmap_...
    └── ...
```

### Metrics CSV Columns

| Column | Description |
|--------|-------------|
| `T_exc_K` | Excess noise temperature |
| `T_exc_error_K` | Error in T_exc |
| `T_sys_K` | System noise temperature |
| `T_sys_error_K` | Error in T_sys |
| `quantum_efficiency` | QE (0-1, higher is better) |
| `quantum_efficiency_error` | Error in QE (propagated from T_exc) |
| `fit_r2` | R² of linear fit |
| `fit_slope` | Fit slope (W/K) |
| `fit_intercept` | Fit intercept (W) |
| `fit_n_points` | Number of temperature points |
| `fit_quality` | 'good'/'acceptable'/'poor' |
| `fit_valid` | Boolean: meets R² threshold |

### Error Bars in Plots

Error bars are automatically displayed when error columns are available. Control via config:

```yaml
plots:
  - type: line
    x: probe_freq_hz
    y: quantum_efficiency
    show_error_bars: true  # default: true
    y_error: quantum_efficiency_error  # auto-detected if omitted
```

## Legacy Format

For legacy data with `metadata.yaml`, use the existing `noise_analysis` package:

```python
# Legacy code continues to work
from noise_analysis import NoiseAnalyzer as LegacyAnalyzer
```

This v2 package is intentionally separate to maintain clean code boundaries.

## Development

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Format code
black noise_analysis_v2/

# Lint
flake8 noise_analysis_v2/
```

## License

MIT
