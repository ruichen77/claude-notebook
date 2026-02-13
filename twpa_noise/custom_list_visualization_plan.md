# Custom List Mode Visualization Plan for noise_analysis_v2

## Executive Summary

This document outlines the modifications needed to enhance `noise_analysis_v2` to effectively visualize noise temperature measurements from the `twpa_noise_measurement` package's **custom_list mode**. Custom_list mode uses specific pump parameter combinations (not Cartesian products) selected for equal gain at target signal frequencies.

---

## 1. Understanding Custom List Mode Data

### 1.1 Data Structure Differences

**Cartesian Mode (Current):**
- Regular grid: All combinations of pump_power × pump_freq
- Example: 5 powers × 10 frequencies = 50 configs
- Natural for heatmaps and 2D parameter sweeps

**Custom List Mode (New):**
- Arbitrary pump parameter combinations
- Example: 46 configs with varying (pump_power, pump_freq) pairs
- Each config selected for ~equal gain at specific signal frequency
- Not grid-based → requires different visualization approaches

### 1.2 Key Characteristics

From `CUSTOM_LIST_MODE.md`:
- Temperature sweep applied to ALL custom configurations
- Each config has unique (pump_power, pump_freq, pump_phase) combination
- Configs grouped by target gain level (e.g., "15 dB gain at 6.5 GHz")
- Total measurements: N_temps × N_configs × N_probe_freqs

### 1.3 experiment_index.csv Format

Custom_list data uses the same `experiment_index.csv` format:
```
temperature_k, probe_freq_hz, pump1_power_dbm, pump1_freq_hz, pump1_phase_deg,
signal_mag_dbm, background_mag_dbm, gain_db, ...
```

**Key difference:** No regular grid structure in pump parameters.

---

## 2. Current Visualization Capabilities

### 2.1 Existing Plot Types (from analysis)

**Standard Plots** (`standard_plots.py`):
- Line plots: QE vs frequency, grouped by pump_power
- Heatmaps: QE vs pump1_power × pump2_power (requires grid)
- Fit quality: R² vs frequency

**Aggregated Plots** (`aggregated_plots.py`):
- Scatter with error bars
- Averages over specified dimensions (e.g., frequency)
- Example: Avg QE vs Avg Gain
- Supports multi-device comparison

**Facet Plots** (`facet_plots.py`):
- Subplots by parameter (e.g., frequency)
- Color by another parameter (e.g., pump_power)
- Linear fit overlays

**Diagnostic Plots** (`diagnostic_plots.py`):
- P_out vs T_in linear fits
- Faceted by frequency, colored by pump_power
- Toggle between "Fit View" and "Data View"

### 2.2 Current Limitations for Custom List

1. **Heatmaps assume grid structure** → Won't work for arbitrary configs
2. **Grouping by pump_power** → Custom list has varying pump_power per config
3. **No config_index tracking** → Can't identify/compare specific configurations
4. **No equal-gain grouping** → Can't visualize configs with same target gain
5. **Limited config-level metadata** → Can't show which configs were selected together

---

## 3. Visualization Gaps for Custom List Mode

### 3.1 Missing Plot Types

1. **Config-indexed plots**
   - QE vs config_index (sequential configuration number)
   - T_sys vs config_index
   - Allows seeing performance across all custom configs

2. **Gain-grouped comparisons**
   - Compare QE for configs with equal target gain
   - Show spread in noise performance at same gain level
   - Critical for understanding gain-noise tradeoff

3. **Pump parameter scatter plots**
   - QE vs pump_power (scatter, not line)
   - QE vs pump_freq (scatter)
   - Color by gain_db or config_index
   - Shows relationship without assuming grid

4. **Config-specific detail views**
   - Individual config performance across frequencies
   - Temperature sweep results for specific config
   - Useful for debugging/understanding outliers

5. **Multi-config comparison**
   - Overlay multiple configs on same plot
   - Compare noise metrics for different pump combinations
   - Identify best-performing configs

### 3.2 Missing Metadata Support

1. **Config identification**
   - Need `config_index` column in data
   - Need `target_gain_db` metadata
   - Need `config_group` for equal-gain sets

2. **Config selection context**
   - Which S21 measurement generated these configs?
   - What was the target gain tolerance?
   - What signal frequency was used for selection?

---

## 4. Proposed Enhancements

### 4.1 Data Loader Modifications

**File:** `data_loader.py`

**Changes:**
1. Add `config_index` detection/generation
   - If not in experiment_index.csv, generate from unique pump combinations
   - Preserve order from custom_configs list

2. Add metadata extraction
   - Read `target_gain_db` from experiment_config.yaml if available
   - Detect config groupings (equal-gain sets)

3. Add custom_list mode detection
   - Flag dataset as custom_list vs cartesian
   - Store in DataFrame metadata

**New function:**
```python
def detect_sweep_mode(df: pd.DataFrame) -> str:
    """
    Detect if data is from cartesian or custom_list sweep.
    
    Returns: 'cartesian', 'custom_list', or 'unknown'
    """
    # Check for regular grid in pump parameters
    # If irregular → custom_list
```

### 4.2 New Plot Types

#### 4.2.1 Config Index Plots

**File:** `plotting/custom_list_plots.py` (NEW)

**Function:** `plot_metric_vs_config_index()`
```python
def plot_metric_vs_config_index(
    df: pd.DataFrame,
    metric: str = 'quantum_efficiency',
    group_by: str = 'probe_freq_hz',
    color_by: str = 'gain_db',
    output_folder: Path = None,
    formats: List[str] = None
) -> List[str]:
    """
    Plot noise metric vs configuration index.
    
    Shows performance across all custom configurations.
    Useful for identifying best/worst configs.
    """
```

**Features:**
- X-axis: config_index (1, 2, 3, ...)
- Y-axis: QE, T_sys, T_exc, etc.
- Group by: probe_freq_hz (separate lines)
- Color by: gain_db (gradient)
- Hover: Show full pump parameters

#### 4.2.2 Gain-Grouped Comparison

**Function:** `plot_metric_by_gain_group()`
```python
def plot_metric_by_gain_group(
    df: pd.DataFrame,
    metric: str = 'quantum_efficiency',
    gain_tolerance: float = 0.5,  # dB
    output_folder: Path = None,
    formats: List[str] = None
) -> List[str]:
    """
    Compare noise metrics for configs with equal target gain.
    
    Groups configs by gain_db (within tolerance).
    Shows spread in noise performance at same gain level.
    """
```

**Features:**
- Box plots or violin plots per gain group
- X-axis: Gain level (e.g., "14.5-15.5 dB")
- Y-axis: QE distribution
- Shows median, quartiles, outliers
- Facet by probe_freq_hz

#### 4.2.3 Pump Parameter Scatter

**Function:** `plot_metric_vs_pump_scatter()`
```python
def plot_metric_vs_pump_scatter(
    df: pd.DataFrame,
    x: str = 'pump1_power_dbm',
    y: str = 'quantum_efficiency',
    color_by: str = 'gain_db',
    size_by: str = None,
    output_folder: Path = None,
    formats: List[str] = None
) -> List[str]:
    """
    Scatter plot of metric vs pump parameter.
    
    No grid assumption - works for arbitrary configs.
    """
```

**Features:**
- Scatter (not line) plot
- Color gradient by gain or frequency
- Optional size by another metric
- Facet by probe_freq_hz
- Hover shows config_index and all params

#### 4.2.4 Config Detail View

**Function:** `plot_config_detail()`
```python
def plot_config_detail(
    df: pd.DataFrame,
    config_indices: List[int],
    output_folder: Path = None,
    formats: List[str] = None
) -> List[str]:
    """
    Detailed view of specific configurations.
    
    Shows QE vs frequency for selected configs.
    Useful for comparing best performers.
    """
```

**Features:**
- Multi-panel: one subplot per config
- QE vs probe_freq_hz
- Annotate with pump parameters
- Show gain_db curve
- Temperature sweep overlay

#### 4.2.5 Pump Parameter Heatmap (Interpolated)

**Function:** `plot_interpolated_heatmap()`
```python
def plot_interpolated_heatmap(
    df: pd.DataFrame,
    x: str = 'pump1_power_dbm',
    y: str = 'pump1_freq_hz',
    z: str = 'quantum_efficiency',
    interpolation: str = 'linear',
    output_folder: Path = None,
    formats: List[str] = None
) -> List[str]:
    """
    Interpolated heatmap for non-grid data.
    
    Uses scipy.interpolate.griddata to create smooth surface.
    Shows estimated performance between measured configs.
    """
```

**Features:**
- Interpolate scattered data to regular grid
- Overlay actual measurement points
- Contour lines for gain_db
- Warning about interpolation artifacts

### 4.3 Configuration File Extensions

**File:** `analyzer.py` - `AnalysisConfig` dataclass

**New fields:**
```python
@dataclass
class AnalysisConfig:
    # ... existing fields ...
    
    # Custom list mode settings
    custom_list_mode: bool = False  # Auto-detect if not specified
    config_index_col: str = 'config_index'  # Column name for config ID
    target_gain_db: Optional[float] = None  # Target gain for grouping
    gain_tolerance: float = 0.5  # dB tolerance for gain grouping
    
    # Custom list plot settings
    show_config_index_plots: bool = True
    show_gain_grouped_plots: bool = True
    show_pump_scatter_plots: bool = True
    interpolate_heatmaps: bool = False  # Use interpolation for heatmaps
```

**Example YAML:**
```yaml
# Custom list mode configuration
analysis:
  custom_list_mode: true  # or auto-detect
  target_gain_db: 15.0
  gain_tolerance: 0.5
  
plots:
  # Config index plots
  - type: config_index
    y: quantum_efficiency
    group_by: probe_freq_hz
    color_by: gain_db
    title: "QE vs Configuration Index"
  
  # Gain-grouped comparison
  - type: gain_grouped
    metric: quantum_efficiency
    title: "QE Distribution by Gain Level"
  
  # Pump parameter scatter
  - type: pump_scatter
    x: pump1_power_dbm
    y: quantum_efficiency
    color_by: gain_db
    facet_by: probe_freq_hz
    title: "QE vs Pump Power (Scatter)"
  
  # Config detail view
  - type: config_detail
    config_indices: [1, 5, 10, 20]  # Best performers
    title: "Top Configuration Details"
  
  # Interpolated heatmap (optional)
  - type: interpolated_heatmap
    x: pump1_power_dbm
    y: pump1_freq_hz
    z: quantum_efficiency
    slice_by: [probe_freq_hz]
    title: "QE Heatmap (Interpolated)"
```

### 4.4 Standard Plots Auto-Generation

**File:** `plotting/standard_plots.py`

**Modify:** `generate_standard_plots()`

**Logic:**
```python
def generate_standard_plots(metrics_df: pd.DataFrame, ...):
    # Detect sweep mode
    sweep_mode = detect_sweep_mode(metrics_df)
    
    if sweep_mode == 'custom_list':
        # Generate custom_list-specific plots
        plots.extend([
            {'type': 'config_index', 'y': 'quantum_efficiency', ...},
            {'type': 'gain_grouped', 'metric': 'quantum_efficiency', ...},
            {'type': 'pump_scatter', 'x': 'pump1_power_dbm', ...},
        ])
    else:
        # Generate cartesian plots (existing)
        plots.extend([
            {'type': 'line', 'x': 'probe_freq_hz', ...},
            {'type': 'heatmap', 'x': 'pump1_power_dbm', ...},
        ])
    
    return engine.generate_from_config(metrics_df, plots)
```

---

## 5. Implementation Approach

### 5.1 Phase 1: Data Infrastructure (Week 1)

**Priority: HIGH**

1. **Add config_index support**
   - Modify `data_loader.py` to detect/generate config_index
   - Add to STANDARD_COLUMNS
   - Test with custom_list data

2. **Add sweep mode detection**
   - Implement `detect_sweep_mode()` function
   - Store in DataFrame metadata
   - Add unit tests

3. **Extend AnalysisConfig**
   - Add custom_list fields
   - Update YAML parsing
   - Update example configs

**Deliverables:**
- Modified `data_loader.py`
- New `sweep_mode_detection.py` utility
- Updated `AnalysisConfig` class
- Unit tests

### 5.2 Phase 2: Core Plot Types (Week 2)

**Priority: HIGH**

1. **Create custom_list_plots.py module**
   - Implement `plot_metric_vs_config_index()`
   - Implement `plot_metric_vs_pump_scatter()`
   - Follow existing plotting conventions

2. **Integrate with PlottingEngine**
   - Add 'config_index' and 'pump_scatter' plot types
   - Update `generate_plot()` dispatcher
   - Add to PlotConfig dataclass

3. **Update standard_plots.py**
   - Add auto-detection logic
   - Generate appropriate plots based on sweep_mode

**Deliverables:**
- New `plotting/custom_list_plots.py` (200-300 lines)
- Modified `plotting/engine.py`
- Modified `plotting/standard_plots.py`
- Example plots with test data

### 5.3 Phase 3: Advanced Visualizations (Week 3)

**Priority: MEDIUM**

1. **Gain-grouped plots**
   - Implement `plot_metric_by_gain_group()`
   - Box plots and violin plots
   - Statistical annotations

2. **Config detail views**
   - Implement `plot_config_detail()`
   - Multi-panel layouts
   - Parameter annotations

3. **Interpolated heatmaps**
   - Implement `plot_interpolated_heatmap()`
   - Add scipy.interpolate dependency
   - Warning overlays for extrapolation

**Deliverables:**
- Extended `custom_list_plots.py` (+200 lines)
- Example gallery
- Documentation

### 5.4 Phase 4: Documentation & Testing (Week 4)

**Priority: HIGH**

1. **Documentation**
   - Update README.md with custom_list examples
   - Create CUSTOM_LIST_VISUALIZATION.md guide
   - Add docstrings to all new functions

2. **Example configs**
   - Create example YAML for custom_list
   - Add to `plotting/` directory
   - Include comments explaining each plot type

3. **Testing**
   - Unit tests for new plot functions
   - Integration tests with real custom_list data
   - Visual regression tests (compare outputs)

**Deliverables:**
- Updated documentation
- Example config files
- Test suite

---

## 6. Technical Specifications

### 6.1 New Module Structure

```
noise_analysis_v2/
├── plotting/
│   ├── __init__.py
│   ├── engine.py (MODIFIED)
│   ├── config.py (MODIFIED)
│   ├── standard_plots.py (MODIFIED)
│   ├── custom_list_plots.py (NEW)
│   ├── sweep_mode_detection.py (NEW)
│   └── examples/
│       ├── config_custom_list_example.yaml (NEW)
│       └── config_cartesian_example.yaml (EXISTING)
```

### 6.2 Key Functions Summary

| Function | Module | Purpose | Priority |
|----------|--------|---------|----------|
| `detect_sweep_mode()` | sweep_mode_detection.py | Identify cartesian vs custom_list | HIGH |
| `generate_config_index()` | data_loader.py | Create config_index column | HIGH |
| `plot_metric_vs_config_index()` | custom_list_plots.py | QE vs config number | HIGH |
| `plot_metric_vs_pump_scatter()` | custom_list_plots.py | Scatter plots for pump params | HIGH |
| `plot_metric_by_gain_group()` | custom_list_plots.py | Box plots by gain level | MEDIUM |
| `plot_config_detail()` | custom_list_plots.py | Individual config analysis | MEDIUM |
| `plot_interpolated_heatmap()` | custom_list_plots.py | Smooth heatmaps | LOW |

### 6.3 Dependencies

**New:**
- `scipy` (for interpolation) - already in requirements

**Existing:**
- `pandas`, `numpy`, `matplotlib`, `plotly`

---

## 7. Example Usage

### 7.1 CLI Usage

```bash
# Auto-detect sweep mode and generate appropriate plots
noise-analysis-v2 run custom_list_config.yaml

# Force custom_list mode
noise-analysis-v2 run config.yaml --custom-list-mode

# Generate only config_index plots
noise-analysis-v2 plot metrics.csv --plot-type config_index -o ./plots
```

### 7.2 Python API

```python
from noise_analysis_v2 import NoiseAnalyzer, AnalysisConfig
from noise_analysis_v2.plotting import plot_metric_vs_config_index

# Full pipeline
config = AnalysisConfig.from_yaml('custom_list_config.yaml')
config.custom_list_mode = True
analyzer = NoiseAnalyzer(config)
results = analyzer.run()

# Individual plot
from noise_analysis_v2 import DataLoader, compute_noise_metrics

loader = DataLoader()
df = loader.load('/path/to/custom_list_measurement')
metrics = compute_noise_metrics(df)

# Generate config index plot
plot_metric_vs_config_index(
    metrics,
    metric='quantum_efficiency',
    group_by='probe_freq_hz',
    color_by='gain_db',
    output_folder='./plots'
)
```

---

## 8. Validation & Testing

### 8.1 Test Data Requirements

1. **Real custom_list measurement**
   - From twpa_noise_measurement package
   - ~46 configs with equal gain
   - Multiple probe frequencies
   - Temperature sweep

2. **Synthetic test data**
   - Generate programmatically
   - Known ground truth
   - Edge cases (single config, missing data)

### 8.2 Validation Criteria

**Functional:**
- [ ] Config index correctly assigned
- [ ] Sweep mode correctly detected
- [ ] All plot types generate without errors
- [ ] Hover text shows correct parameters
- [ ] Gain grouping works with tolerance

**Visual:**
- [ ] Config index plots show clear trends
- [ ] Scatter plots don't assume grid
- [ ] Gain-grouped plots show distributions
- [ ] Colors/markers distinguish configs
- [ ] Legends are readable

**Performance:**
- [ ] Plotting time < 5s for 1000 configs
- [ ] Memory usage reasonable for large datasets
- [ ] Interpolation doesn't hang

---

## 9. Migration Path

### 9.1 Backward Compatibility

**Guarantee:** Existing cartesian mode configs continue to work unchanged.

**Strategy:**
1. Auto-detect sweep mode (default behavior)
2. If cartesian → use existing plot types
3. If custom_list → use new plot types
4. User can override with `custom_list_mode: true/false`

### 9.2 Deprecation Plan

**None required** - This is additive functionality.

---

## 10. Future Enhancements

### 10.1 Short-term (3-6 months)

1. **Config optimization visualization**
   - Show Pareto front (gain vs QE)
   - Highlight optimal configs
   - Export optimal config list

2. **Interactive config selection**
   - Click on plot to see config details
   - Filter by performance thresholds
   - Export selected configs to YAML

3. **Comparison with Cartesian**
   - Overlay custom_list results on full Cartesian sweep
   - Show which configs were selected
   - Quantify measurement time savings

### 10.2 Long-term (6-12 months)

1. **Machine learning integration**
   - Predict QE for untested configs
   - Suggest next configs to measure
   - Active learning loop

2. **Multi-device comparison**
   - Compare custom_list results across devices
   - Identify device-specific trends
   - Batch analysis

3. **Real-time monitoring**
   - Live plots during measurement
   - Early stopping if performance degrades
   - Adaptive config selection

---

## 11. Success Metrics

### 11.1 Quantitative

- [ ] All 5 core plot types implemented
- [ ] 100% backward compatibility maintained
- [ ] <5s plot generation time
- [ ] >90% test coverage for new code

### 11.2 Qualitative

- [ ] Users can identify best configs visually
- [ ] Gain-noise tradeoffs are clear
- [ ] Config selection rationale is understandable
- [ ] Documentation is comprehensive

---

## 12. Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Config index not in data | High | High | Auto-generate from pump params |
| Interpolation artifacts | Medium | Medium | Add warnings, make optional |
| Performance with 1000+ configs | Low | Medium | Optimize plotting, add sampling |
| User confusion about modes | Medium | Low | Clear documentation, auto-detect |
| Breaking existing workflows | Low | High | Extensive testing, backward compat |

---

## 13. Timeline Summary

| Phase | Duration | Deliverables | Dependencies |
|-------|----------|--------------|--------------|
| Phase 1: Data Infrastructure | 1 week | Config index, sweep detection | None |
| Phase 2: Core Plots | 1 week | Config index & scatter plots | Phase 1 |
| Phase 3: Advanced Viz | 1 week | Gain groups, detail views | Phase 2 |
| Phase 4: Docs & Testing | 1 week | Documentation, tests | Phase 3 |
| **Total** | **4 weeks** | Full custom_list support | - |

---

## 14. Conclusion

This plan provides a comprehensive roadmap for adding custom_list mode visualization to `noise_analysis_v2`. The phased approach ensures:

1. **Backward compatibility** - Existing workflows unaffected
2. **Incremental value** - Core features delivered early
3. **Extensibility** - Foundation for future enhancements
4. **User-friendly** - Auto-detection and clear documentation

The key innovation is recognizing that custom_list data requires **scatter-based** and **config-indexed** visualizations rather than grid-based heatmaps. By adding these plot types while maintaining the existing architecture, we enable comprehensive analysis of targeted pump parameter measurements.

**Next Steps:**
1. Review and approve this plan
2. Set up development branch
3. Begin Phase 1 implementation
4. Schedule weekly progress reviews