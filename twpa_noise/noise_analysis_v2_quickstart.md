# noise_analysis_v2 — Quick Start (Fast Track)

Run noise temperature analysis on completed measurements. Package is installed on **timtam** at `/nas-data0/systems/fridge36B/Ruichen_library/noise_analysis_package2/`. Local source: `~/repos/noise_analysis_v2/`.

## Workflow

1. Write a YAML config locally
2. SCP config to timtam benchmark folder
3. Run `noise-analysis-v2 run config.yaml` on timtam
4. (Optional) SCP `noise_metrics.csv` back for local inspection

## Config Template — Standard (Cartesian) Mode

```yaml
thru_paths:
  my_thru: /path/to/noise_test1_nestedthru_reference

data_sources:
  - path: /path/to/noise_test1_DEVICE_noise
    label: "DEVICE_LABEL"
    thru_path_ref: my_thru

analysis:
  pump_freq_col: pump1_freq_hz
  idler_gain: 0
  r2_threshold: 0.5

plots:
  - type: line
    x: probe_freq_hz
    y: quantum_efficiency
    group_by: [pump1_power_dbm]
    title: "QE vs Frequency"
  - type: line
    x: probe_freq_hz
    y: T_sys_K
    group_by: [pump1_power_dbm]
    title: "T_sys vs Frequency"
  - type: heatmap
    x: pump1_power_dbm
    y: pump1_freq_hz
    z: quantum_efficiency
    slice_by: [probe_freq_hz]
    title: "QE Heatmap"
  - type: line
    x: probe_freq_hz
    y: fit_r2
    group_by: [pump1_power_dbm]
    title: "Fit Quality (R²)"

output:
  folder: /path/to/analysis_output_DEVICE
  formats: [html, png]

diagnostic_plots:
  r2_threshold: 0.7
  require_positive_texc: true
```

## Config Template — Custom List Mode

For measurements using `sweep.mode: custom_list` (e.g., equal-gain configs):

```yaml
thru_paths:
  my_thru: /path/to/noise_test1_nestedthru_reference

data_sources:
  - path: /path/to/noise_test1_DEVICE_equal_gain_noise
    label: "DEVICE Equal Gain"
    thru_path_ref: my_thru

analysis:
  pump_freq_col: pump1_freq_hz
  idler_gain: 0
  r2_threshold: 0.5
  custom_list_mode: true
  target_gain_db: 15.0
  gain_tolerance: 0.5

plots:
  # Custom list specific
  - type: gain_grouped
    metric: quantum_efficiency
    title: "QE Distribution by Gain Level"
  - type: gain_grouped
    metric: T_sys_K
    title: "T_sys Distribution by Gain Level"
  - type: pump_scatter
    x: pump1_power_dbm
    y: quantum_efficiency
    color_by: gain_db
    facet_by: probe_freq_hz
    title: "QE vs Pump Power"
  - type: pump_scatter
    x: pump1_freq_hz
    y: quantum_efficiency
    color_by: gain_db
    facet_by: probe_freq_hz
    title: "QE vs Pump Frequency"
  - type: pump_scatter
    x: pump1_power_dbm
    y: T_sys_K
    color_by: gain_db
    title: "T_sys vs Pump Power"
  - type: aggregated
    x: gain_db
    y: quantum_efficiency
    agg_over: temperature_k
    error_type: sem
    title: "Average QE vs Gain"
  - type: aggregated
    x: gain_db
    y: T_sys_K
    agg_over: temperature_k
    error_type: sem
    title: "Average T_sys vs Gain"
  # Standard
  - type: line
    x: probe_freq_hz
    y: quantum_efficiency
    group_by: [pump1_power_dbm]
    title: "QE vs Probe Frequency"
  - type: line
    x: probe_freq_hz
    y: fit_r2
    group_by: [pump1_power_dbm]
    title: "Fit Quality (R²)"

output:
  folder: /path/to/analysis_output_DEVICE_equal_gain
  formats: [html, png]

diagnostic_plots:
  r2_threshold: 0.7
  require_positive_texc: true
```

**Note:** `config_index` plot type exists but has a bug where the column doesn't propagate to the metrics DataFrame. Use `gain_grouped` and `pump_scatter` as alternatives.

## CLI Commands

```bash
# Full pipeline (most common)
noise-analysis-v2 run config.yaml

# Quick data inspection
noise-analysis-v2 load /path/to/measurement

# Direct compute (no plots)
noise-analysis-v2 compute /path/to/measurement --thru /path/to/thru -o metrics.csv

# Plot from existing metrics
noise-analysis-v2 plot metrics.csv -o ./plots
```

## Output Files

```
analysis_output/
├── noise_metrics.csv      # T_exc, T_sys, QE, fit stats per config/freq
├── raw_data.csv           # All raw measurements with thru-recomputed gain
├── thru_reference.csv     # Thru reference data
├── pout_vs_tin.html/.png  # Diagnostic P_out vs T_in linear fits
├── *.html                 # Interactive Plotly plots
└── *.png                  # Static Matplotlib plots
```

## Metrics CSV Key Columns

| Column | Description |
|--------|-------------|
| `quantum_efficiency` | QE (0-1, higher = better) |
| `T_sys_K` | System noise temperature |
| `T_exc_K` | Excess noise temperature |
| `gain_db` | Gain at coldest temperature |
| `fit_r2` | R² of Y-factor linear fit |
| `fit_valid` | Meets R² threshold |

## Quick Local Inspection

```python
import pandas as pd
df = pd.read_csv('noise_metrics.csv')
valid = df[df['fit_valid']]
print(f'QE: {valid["quantum_efficiency"].mean():.3f} +/- {valid["quantum_efficiency"].std():.3f}')
print(f'Best QE: {valid["quantum_efficiency"].max():.3f}')
best = valid.nlargest(5, 'quantum_efficiency')[['probe_freq_hz','pump1_power_dbm','pump1_freq_hz','gain_db','quantum_efficiency','T_sys_K','fit_r2']]
print(best.to_string(index=False))
```

## Paths (Feb 2026 cooldown)

- **Data**: `timtam:/nas-data1/systems/fridge36B/20260202_RANv401_RANv402/benchmark/`
- **Thru**: `.../noise_test1_nestedthru_reference`
- **Measurements**: `noise_test1_<DEVICE>_noise` or `noise_test1_<DEVICE>_equal_gain_noise`
- **Analysis output**: `analysis_<DEVICE>_<suffix>/`
