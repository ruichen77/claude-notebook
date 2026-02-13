# Custom List Sweep Mode

## Overview

The custom list sweep mode allows you to measure noise temperature at **specific pump parameter configurations** instead of measuring all Cartesian product combinations. This is useful when you want to test targeted parameter sets based on prior knowledge or optimization results.

## Key Features

- **Targeted measurements**: Define exact pump parameter combinations to measure
- **Temperature sweep**: Temperature sweep applies to ALL custom configurations
- **Probe frequency sweep**: All probe frequencies are measured for each configuration
- **Backward compatible**: Existing configs continue to work with default `cartesian` mode
- **Flexible**: Works with both single and cascaded TWPA configurations

## Configuration Format

### Basic Structure

```yaml
sweep:
  mode: custom_list  # Enable custom list mode
  
  loop_order:
    - temperature  # Temperature is outer loop
  
  axes:
    temperature: [0.2, 0.3, 0.4]  # Temperature sweep
  
  custom_configs:
    # List of specific pump parameter combinations
    - pump1_power: -3.0
      pump1_frequency: 8.42e9
      pump1_phase: 0
    
    - pump1_power: -2.5
      pump1_frequency: 8.40e9
      pump1_phase: 90
```

### Measurement Order

For the above configuration:
1. T=0.2K → Config 1 → all probe frequencies
2. T=0.2K → Config 2 → all probe frequencies
3. T=0.3K → Config 1 → all probe frequencies
4. T=0.3K → Config 2 → all probe frequencies
5. T=0.4K → Config 1 → all probe frequencies
6. T=0.4K → Config 2 → all probe frequencies

**Total measurements**: 3 temps × 2 configs × N probe_freqs

## Examples

### Single TWPA

See [`noise_measurement/examples/config_custom_list.yaml`](noise_measurement/examples/config_custom_list.yaml)

```yaml
sweep:
  mode: custom_list
  loop_order:
    - temperature
  axes:
    temperature: [0.2, 0.3, 0.4]
  
  custom_configs:
    - pump1_power: -3.0
      pump1_frequency: 8.42e9
      pump1_phase: 0
    
    - pump1_power: -2.5
      pump1_frequency: 8.40e9
      pump1_phase: 90
    
    - pump1_power: -2.0
      pump1_frequency: 8.38e9
      pump1_phase: 45
```

### Cascaded TWPA

See [`noise_measurement/examples/config_custom_list_cascaded.yaml`](noise_measurement/examples/config_custom_list_cascaded.yaml)

```yaml
sweep:
  mode: custom_list
  loop_order:
    - temperature
  axes:
    temperature: [0.2, 0.3]
  
  custom_configs:
    - pump1_power: -3.0
      pump1_frequency: 8.42e9
      pump1_phase: 0
      pump2_power: -8.0
      pump2_frequency: 8.39e9
      pump2_phase: 90
    
    - pump1_power: -2.5
      pump1_frequency: 8.42e9
      pump1_phase: 0
      pump2_power: -8.5
      pump2_frequency: 8.39e9
      pump2_phase: 90
```

## Comparison: Cartesian vs Custom List

### Cartesian Mode (Default)

```yaml
sweep:
  mode: cartesian  # or omit (default)
  loop_order:
    - temperature
    - pump1_power
  axes:
    temperature: [0.2, 0.3]
    pump1_power: [-3.0, -2.5, -2.0]
```

**Result**: 2 temps × 3 powers × N probe_freqs = 6N measurements

### Custom List Mode

```yaml
sweep:
  mode: custom_list
  loop_order:
    - temperature
  axes:
    temperature: [0.2, 0.3]
  custom_configs:
    - pump1_power: -3.0
      pump1_frequency: 8.42e9
    - pump1_power: -2.5
      pump1_frequency: 8.40e9
```

**Result**: 2 temps × 2 configs × N probe_freqs = 4N measurements

## Validation Rules

1. **Mode selection**: `mode` must be either `cartesian` or `custom_list`
2. **Custom configs required**: `custom_configs` must be a non-empty list when using `custom_list` mode
3. **Valid parameters**: Only pump parameters allowed in `custom_configs`:
   - `pump1_power`, `pump1_frequency`, `pump1_phase`
   - `pump2_power`, `pump2_frequency`, `pump2_phase` (cascaded only)
   - `probe_phase` (optional)
4. **Temperature in axes**: In `custom_list` mode, only `temperature` is allowed in `axes`
5. **Device type validation**: `pump2_*` parameters only valid for `cascaded_twpa`

## Usage

### Running a Measurement

```bash
# Validate configuration
python -m noise_measurement validate -c config_custom_list.yaml

# Run measurement
python -m noise_measurement run -c config_custom_list.yaml

# Dry run (no hardware)
python -m noise_measurement run -c config_custom_list.yaml --dry-run
```

### Estimating Duration

```bash
python -m noise_measurement estimate -c config_custom_list.yaml
```

## Implementation Details

- **Class**: `CustomListSweep` in `noise_measurement/sweep.py`
- **Interface**: Same as `ParameterSweep` (generates `MeasurementPoint` objects)
- **Factory pattern**: `core.py` selects sweep class based on `mode` config
- **No downstream changes**: All other modules work unchanged

## Migration from Cartesian Mode

Existing configurations continue to work without modification. To convert:

1. Add `mode: custom_list` to sweep section
2. Move `temperature` to `axes`, remove other parameters
3. Add `custom_configs` list with desired pump combinations

## Benefits

- **Efficiency**: Measure only relevant parameter combinations
- **Targeted**: Test specific configurations from optimization or theory
- **Flexible**: Mix different pump parameters in each configuration
- **Time-saving**: Reduce measurement time by avoiding unnecessary combinations

## Limitations

- Temperature must be swept for all configurations (cannot specify per-config)
- Probe frequencies are always swept (cannot specify per-config)
- Cannot mix cartesian and custom list modes in single measurement

## Future Enhancements

Potential future additions:
- Per-configuration temperature specification
- Per-configuration probe frequency lists
- CSV file import for large configuration lists
- Configuration generation from optimization results