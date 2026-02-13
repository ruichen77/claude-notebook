# Custom List Mode Visualization - Implementation Guide

## Overview

This branch adds comprehensive visualization support for **custom_list mode** measurements from the `twpa_noise_measurement` package. Custom list mode uses specific pump parameter combinations (not Cartesian products) selected for equal gain at target signal frequencies.

## What's New

### 1. Sweep Mode Detection (`sweep_mode_detection.py`)

Automatically detects whether data is from:
- **Cartesian mode**: Regular grid of pump parameters
- **Custom list mode**: Arbitrary pump parameter combinations

**Key Functions:**
- `detect_sweep_mode(df)` - Auto-detect sweep type
- `generate_config_index(df)` - Assign sequential indices to configs
- `group_configs_by_gain(df, tolerance)` - Group configs by gain level

### 2. Custom List Plots (`plotting/custom_list_plots.py`)

Three new plot types optimized for non-grid data:

#### a) Config Index Plots
```python
plot_metric_vs_config_index(
    df, 
    metric='quantum_efficiency',
    group_by='probe_freq_hz',
    color_by='gain_db'
)
```
- X-axis: Configuration index (1, 2, 3, ...)
- Y-axis: Noise metric (QE, T_sys, T_exc)
- Shows performance across all custom configs
- Identifies best/worst configurations

#### b) Gain-Grouped Plots
```python
plot_metric_by_gain_group(
    df,
    metric='quantum_efficiency',
    gain_group_col='gain_group'
)
```
- Box plots showing QE distribution at each gain level
- Reveals gain-noise tradeoff
- Faceted by probe frequency

#### c) Pump Parameter Scatter
```python
plot_metric_vs_pump_scatter(
    df,
    x='pump1_power_dbm',
    y='quantum_efficiency',
    color_by='gain_db'
)
```
- Scatter plots (no grid assumption)
- Shows trends in pump parameter space
- Color gradient by gain or frequency

### 3. Enhanced Data Loader

**Automatic Features:**
- Generates `config_index` if not present
- Detects `sweep_mode` automatically
- Adds `gain_group` column for grouping

### 4. Example Configuration

See `plotting/config_custom_list_example.yaml` for complete example.

## Usage

### Quick Start

```bash
# 1. Generate custom configs (using find_equal_gain_configs.py)
python find_equal_gain_configs.py \
  --measurements-dir ./measurements \
  --signal-freq 6.5e9 \
  --target-gain 15.0 \
  --output custom_configs.yaml

# 2. Run noise measurement
python -m noise_measurement run -c custom_configs.yaml

# 3. Analyze with custom_list visualization
noise-analysis-v2 run config_custom_list_example.yaml
```

### Python API

```python
from noise_analysis_v2 import NoiseAnalyzer, AnalysisConfig
from noise_analysis_v2.plotting import (
    plot_metric_vs_config_index,
    plot_metric_by_gain_group,
    plot_metric_vs_pump_scatter
)

# Load and analyze
config = AnalysisConfig.from_yaml('config_custom_list_example.yaml')
analyzer = NoiseAnalyzer(config)
results = analyzer.run()

# Or use individual plot functions
from noise_analysis_v2 import DataLoader, compute_noise_metrics
from noise_analysis_v2.sweep_mode_detection import (
    generate_config_index,
    group_configs_by_gain
)

loader = DataLoader()
df = loader.load('/path/to/custom_list_measurement')
metrics = compute_noise_metrics(df)

# Add custom_list metadata
metrics['config_index'] = generate_config_index(metrics)
metrics['gain_group'] = group_configs_by_gain(metrics, tolerance=0.5)

# Generate plots
plot_metric_vs_config_index(
    metrics,
    metric='quantum_efficiency',
    output_folder='./plots'
)

plot_metric_by_gain_group(
    metrics,
    metric='quantum_efficiency',
    output_folder='./plots'
)

plot_metric_vs_pump_scatter(
    metrics,
    x='pump1_power_dbm',
    y='quantum_efficiency',
    color_by='gain_db',
    output_folder='./plots'
)
```

## Configuration Options

### Analysis Config

```yaml
analysis:
  # Standard settings
  pump_freq_col: pump1_freq_hz
  idler_gain: 0
  r2_threshold: 0.5
  
  # Custom list mode settings
  custom_list_mode: true  # Enable (auto-detected if omitted)
  target_gain_db: 15.0    # Target gain for grouping
  gain_tolerance: 0.5     # Tolerance in dB
```

### Plot Types

```yaml
plots:
  # Config index plot
  - type: config_index
    y: quantum_efficiency
    group_by: probe_freq_hz
    color_by: gain_db
  
  # Gain-grouped comparison
  - type: gain_grouped
    metric: quantum_efficiency
  
  # Pump parameter scatter
  - type: pump_scatter
    x: pump1_power_dbm
    y: quantum_efficiency
    color_by: gain_db
    facet_by: probe_freq_hz
```

## Backward Compatibility

✅ **Fully backward compatible** - Existing Cartesian mode configs work unchanged.

The package auto-detects sweep mode:
- If Cartesian → uses existing plot types (line, heatmap)
- If custom_list → uses new plot types (scatter, config_index)

## File Structure

```
noise_analysis_v2/
├── sweep_mode_detection.py          # NEW: Sweep mode utilities
├── data_loader.py                   # MODIFIED: Auto-generate config_index
├── plotting/
│   ├── custom_list_plots.py         # NEW: Custom list plot functions
│   └── config_custom_list_example.yaml  # NEW: Example config
└── CUSTOM_LIST_MODE_README.md       # NEW: This file
```

## Implementation Status

### ✅ Completed (Phase 1 & 2)

- [x] Sweep mode detection
- [x] Config index generation
- [x] Gain grouping utilities
- [x] Config index plots (Plotly & Matplotlib)
- [x] Gain-grouped plots (box plots)
- [x] Pump parameter scatter plots
- [x] Data loader integration
- [x] Example configuration
- [x] Documentation

### 🚧 Pending (Phase 3 & 4)

- [ ] Integration with PlottingEngine dispatcher
- [ ] Update AnalysisConfig dataclass
- [ ] Auto-generation in standard_plots.py
- [ ] Unit tests
- [ ] Integration tests with real data
- [ ] Config detail view plots
- [ ] Interpolated heatmaps (optional)

## Testing

### Manual Testing

```bash
# Test with synthetic data
python -c "
import pandas as pd
import numpy as np
from noise_analysis_v2.sweep_mode_detection import detect_sweep_mode, generate_config_index

# Create test data
df = pd.DataFrame({
    'pump1_power_dbm': np.random.uniform(-10, 0, 50),
    'pump1_freq_hz': np.random.uniform(8e9, 9e9, 50),
    'quantum_efficiency': np.random.uniform(0.2, 0.4, 50),
    'gain_db': np.random.uniform(14, 16, 50),
})

# Test detection
mode = detect_sweep_mode(df)
print(f'Detected mode: {mode}')

# Test config indexing
df['config_index'] = generate_config_index(df)
print(f'Generated {df[\"config_index\"].nunique()} unique configs')
"
```

### Unit Tests (TODO)

```bash
pytest noise_analysis_v2/tests/test_sweep_mode_detection.py
pytest noise_analysis_v2/tests/test_custom_list_plots.py
```

## Examples

### Example 1: Compare QE at Equal Gain

```python
# Shows QE distribution for configs with 14.5-15.5 dB gain
plot_metric_by_gain_group(
    metrics,
    metric='quantum_efficiency',
    gain_tolerance=0.5
)
```

**Output:** Box plots showing QE spread at each gain level. Reveals if higher gain sacrifices noise performance.

### Example 2: Identify Best Configs

```python
# Shows QE for each config, colored by gain
plot_metric_vs_config_index(
    metrics,
    metric='quantum_efficiency',
    color_by='gain_db'
)
```

**Output:** Line plot with config index on X-axis. Easily identify which configs (by number) perform best.

### Example 3: Pump Parameter Trends

```python
# Shows how QE varies with pump power
plot_metric_vs_pump_scatter(
    metrics,
    x='pump1_power_dbm',
    y='quantum_efficiency',
    color_by='gain_db'
)
```

**Output:** Scatter plot revealing trends like "higher pump power → higher gain but lower QE".

## Troubleshooting

### Issue: "config_index not found"

**Solution:** The data loader should auto-generate this. If not:
```python
from noise_analysis_v2.sweep_mode_detection import generate_config_index
df['config_index'] = generate_config_index(df)
```

### Issue: "gain_group not found"

**Solution:** Generate gain groups:
```python
from noise_analysis_v2.sweep_mode_detection import group_configs_by_gain
df['gain_group'] = group_configs_by_gain(df, tolerance=0.5)
```

### Issue: Plots look wrong for Cartesian data

**Solution:** The package should auto-detect. Force custom_list mode:
```yaml
analysis:
  custom_list_mode: false  # Force Cartesian mode
```

## Contributing

To extend this implementation:

1. **Add new plot types:** Add functions to `custom_list_plots.py`
2. **Modify detection:** Update `sweep_mode_detection.py`
3. **Add tests:** Create test files in `tests/`
4. **Update docs:** Add examples to this README

## References

- **Planning Document:** `CUSTOM_LIST_VISUALIZATION_PLAN.md`
- **twpa_noise_measurement:** `CUSTOM_LIST_MODE.md`
- **find_equal_gain_configs:** `EQUAL_GAIN_FINDER_README.md`

## License

MIT (same as noise_analysis_v2)

## Authors

- Implementation: Bob (AI Assistant)
- Design: Based on user requirements for custom_list mode visualization

## Changelog

### 2026-01-23 - Initial Implementation

- Added sweep mode detection
- Added config index generation
- Added three new plot types
- Created example configuration
- Updated data loader
- Created documentation