# Multi-Device Overlay Enhancement Summary

## Overview
Enhanced the noise-analysis-package v2 to properly support multi-device comparison plots with distinct visual markers for each device.

## Date
2026-01-22

## Issues Fixed

### 1. **Critical Bug: Device Data Merging**
**Problem:** The `data_source` column was incorrectly included in `EXCLUDE_FROM_GROUPING`, causing all devices to be merged into a single trace with the same color and marker.

**Fix:** Removed `'data_source'` from the `EXCLUDE_FROM_GROUPING` set in `aggregated_plots.py` (line 36).

**Impact:** Devices are now properly separated and displayed as individual traces.

### 2. **Missing Visual Differentiation**
**Problem:** All devices used the same marker shape (circles), making it difficult to distinguish them, especially in grayscale prints.

**Fix:** Added marker shape cycling for both Plotly and Matplotlib:
- **Plotly markers:** circle, square, diamond, cross, x, triangle-up, triangle-down, star, hexagon, pentagon
- **Matplotlib markers:** o, s, D, ^, v, <, >, p, *, h

**Impact:** Each device now has a unique marker shape in addition to unique color.

### 3. **Title Truncation**
**Problem:** Long plot titles were getting cut off in the display.

**Fix:** Implemented `_wrap_title()` function that automatically wraps titles longer than 80 characters:
- Uses `<br>` tags for HTML/Plotly plots
- Uses `\n` for Matplotlib/PNG plots

**Impact:** Long titles now display properly across multiple lines.

## Files Modified

### `twpa_measurement/noise_analysis_package2/noise_analysis_v2/plotting/aggregated_plots.py`

**Changes:**
1. Line 36: Removed `'data_source'` from `EXCLUDE_FROM_GROUPING`
2. Lines 213-247: Added `_wrap_title()` helper function
3. Lines 279-290: Added marker symbol cycling for Plotly plots
4. Lines 303-310: Enhanced marker styling with white outlines for better visibility
5. Lines 349-357: Applied title wrapping to Plotly plots
6. Lines 384-420: Added marker style cycling for Matplotlib plots
7. Lines 439-441: Applied title wrapping to Matplotlib plots

## Usage

### Example Configuration (multi_device_comparison_config.yaml)

```yaml
# Load multiple devices
data_sources:
  - path: /path/to/device1
    label: "HDRP094"
    thru_path_ref: ranv502_thru
    
  - path: /path/to/device2
    label: "RANV502D1"
    thru_path_ref: ranv502_thru

# Create aggregated comparison plots
plots:
  - type: aggregated
    x: gain_db
    y: quantum_efficiency
    agg_func: mean
    agg_over: probe_freq_hz
    error_type: std
    title: "Multi-Device Comparison: Avg QE vs Avg Gain"
```

### Running the Analysis

```bash
cd twpa_measurement/noise_analysis_package2
noise-analysis-v2 run ../multi_device_comparison_config.yaml
```

## Expected Output

### Aggregated Plots
Each aggregated plot will show:
- **Multiple traces:** One per device
- **Unique colors:** Automatically assigned from HSL color space
- **Unique markers:** Different shapes for each device
- **Error bars:** Standard deviation or SEM across aggregated dimension
- **Legend:** Clear device labels
- **Wrapped titles:** Long titles split across multiple lines

### Visual Features
- **HDRP094:** First color (e.g., red), circle markers
- **RANV502D1:** Second color (e.g., cyan), square markers
- Additional devices get triangle, diamond, cross, etc.

## Testing Recommendations

1. **Test with your actual data:**
   ```bash
   cd twpa_measurement/noise_analysis_package2
   noise-analysis-v2 run ../multi_device_comparison_config.yaml
   ```

2. **Verify the output:**
   - Check that both devices appear as separate traces
   - Confirm different colors and marker shapes
   - Verify error bars are displayed
   - Check that long titles wrap properly

3. **Common issues to check:**
   - Ensure both device paths are correct
   - Verify `experiment_index.csv` exists in each device folder
   - Check that `data_source` labels are unique
   - Confirm thru reference paths are valid

## Additional Features Available

### Aggregation Options
- `agg_func`: 'mean', 'median', 'max', 'min'
- `agg_over`: Column(s) to average over (e.g., 'probe_freq_hz')
- `error_type`: 'std', 'sem', 'none'

### Plot Customization
```yaml
plots:
  - type: aggregated
    x: gain_db
    y: quantum_efficiency
    agg_func: mean
    agg_over: probe_freq_hz
    error_type: sem  # Use standard error instead of std dev
    x_range: [0, 20]  # Optional: set axis ranges
    y_range: [0, 1]
    title: "Custom Title"
```

## Backward Compatibility

All changes are backward compatible:
- Single-device configurations work as before
- Existing YAML configs remain valid
- No breaking changes to API

## Future Enhancements (Optional)

If needed, these features could be added:
1. Custom device colors via config
2. Custom marker shapes per device
3. Statistical comparison annotations
4. Device-to-device normalization
5. Performance ranking overlays

## Contact

For issues or questions, refer to the package documentation or contact the maintainer.