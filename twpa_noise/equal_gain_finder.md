# Equal Gain Configuration Finder

This tool helps you find pump parameter combinations that produce equal gain for a given signal frequency in S21 measurements. The output can be directly used to generate `custom_configs` for the `twpa_noise_measurement` package.

## Overview

The script analyzes 2D parameter sweep data (pump frequency vs pump power) and identifies combinations that give the same gain at a specified signal frequency. This is useful for:

- Finding optimal pump settings for consistent amplification
- Generating custom measurement lists for noise characterization
- Exploring the parameter space efficiently

## Installation

The script requires:
- Python 3.7+
- numpy
- pandas
- pyyaml

Install dependencies:
```bash
pip install numpy pandas pyyaml
```

## Usage

### Basic Usage - Scan All Gains

Scan all pump configurations and see what gains are available at a specific signal frequency:

```bash
python find_equal_gain_configs.py \
    --measurements-dir /path/to/measurements \
    --signal-freq 6.5e9
```

This will print a summary table showing all pump power/frequency combinations and their corresponding gains.

### Filter by Target Gain

Find configurations that produce a specific gain (e.g., 10 dB ± 0.5 dB):

```bash
python find_equal_gain_configs.py \
    --measurements-dir /path/to/measurements \
    --signal-freq 6.5e9 \
    --target-gain 10.0 \
    --tolerance 0.5
```

### Export to CSV

Save all configurations to a CSV file for further analysis:

```bash
python find_equal_gain_configs.py \
    --measurements-dir /path/to/measurements \
    --signal-freq 7.0e9 \
    --output-csv gains_at_7GHz.csv
```

### Generate YAML for twpa_noise_measurement

Create a YAML configuration file with `custom_configs` format:

```bash
python find_equal_gain_configs.py \
    --measurements-dir /path/to/measurements \
    --signal-freq 6.5e9 \
    --target-gain 12.0 \
    --tolerance 0.3 \
    --output-yaml custom_configs.yaml \
    --phase 0.0 \
    --temperatures 0.2 0.3 0.4
```

This generates a YAML file that can be merged into your measurement configuration.

## Example Workflow

### Step 1: Explore Available Gains

First, scan your measurement data to see what gains are available:

```bash
python find_equal_gain_configs.py \
    --measurements-dir /Users/ruichenzhao/Desktop/S21_analyzer_data/parameter_sweeps_2D/RANV502D1/PUMP01_freq_8.0G-8.6G_vs_PUMP01_pwr_-12.0--7.0_20260120_165857/measurements \
    --signal-freq 6.5e9 \
    --output-csv all_gains_6p5GHz.csv
```

Output:
```
================================================================================
Found 1071 pump configurations at signal frequency 6.500 GHz
================================================================================
Pump Power (dBm)     Pump Freq (GHz)      Gain (dB)      
--------------------------------------------------------------------------------
-7.00                8.550                15.23          
-7.25                8.539                14.87          
-7.50                8.528                14.52          
...
--------------------------------------------------------------------------------
Gain statistics:
  Mean: -35.42 dB
  Std:  5.23 dB
  Min:  -55.84 dB
  Max:  15.23 dB
================================================================================
```

### Step 2: Select Target Gain

Based on the output, choose a target gain and find matching configurations:

```bash
python find_equal_gain_configs.py \
    --measurements-dir /Users/ruichenzhao/Desktop/S21_analyzer_data/parameter_sweeps_2D/RANV502D1/PUMP01_freq_8.0G-8.6G_vs_PUMP01_pwr_-12.0--7.0_20260120_165857/measurements \
    --signal-freq 6.5e9 \
    --target-gain 12.0 \
    --tolerance 0.5 \
    --output-yaml equal_gain_configs.yaml \
    --temperatures 0.2 0.3 0.4
```

### Step 3: Use in Noise Measurement

The generated YAML file contains a `sweep` section that you can copy into your measurement configuration:

```yaml
sweep:
  mode: custom_list
  loop_order:
  - temperature
  axes:
    temperature:
    - 0.2
    - 0.3
    - 0.4
  custom_configs:
  - pump1_power: -8.5
    pump1_frequency: 8440000000.0
    pump1_phase: 0.0
  - pump1_power: -8.75
    pump1_frequency: 8429000000.0
    pump1_phase: 0.0
  # ... more configurations
```

## Command-Line Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--measurements-dir` | Yes | Path to measurements directory containing subdirectories with S21 data |
| `--signal-freq` | Yes | Signal frequency in Hz (e.g., `6.5e9` for 6.5 GHz) |
| `--target-gain` | No | Target gain in dB for filtering |
| `--tolerance` | No | Gain tolerance in dB (default: 0.5) |
| `--reference-s21` | No | Reference S21 value in dB for gain calculation |
| `--output-csv` | No | Output CSV file path for all configurations |
| `--output-yaml` | No | Output YAML file path with custom_configs format |
| `--phase` | No | Pump phase in degrees for YAML output (default: 0.0) |
| `--temperatures` | No | Temperature points for YAML output (default: 0.2 0.3 0.4) |

## Data Structure Requirements

The script expects measurements organized as:

```
measurements/
├── 2026-01-20_16.59.03_..._PUMP01_pwr_-12_00_sweep_PUMP01_freq/
│   └── data/
│       ├── sweep_PUMP01_freq_setValTo_8000000000.0_param_S21_magnitude_phase.csv
│       ├── sweep_PUMP01_freq_setValTo_8011000000.0_param_S21_magnitude_phase.csv
│       └── ...
├── 2026-01-20_17.00.08_..._PUMP01_pwr_-11_75_sweep_PUMP01_freq/
│   └── data/
│       └── ...
└── ...
```

Each CSV file should contain columns:
- `f`: Frequency in Hz
- `S21_magnitude`: S21 magnitude in dB
- `S21_phase`: S21 phase in degrees

## Gain Calculation

The gain is calculated as:
- If `--reference-s21` is provided: `Gain = S21_magnitude - reference_s21`
- Otherwise: `Gain = S21_magnitude` (assumes reference is normalized)

For TWPA measurements, the S21 magnitude in dB directly represents the gain when the pump is on, compared to the unpumped state.

## Tips

1. **Start broad**: First scan without `--target-gain` to see the full range of available gains
2. **Adjust tolerance**: Use tighter tolerance (e.g., 0.2 dB) for more uniform gain, or looser (e.g., 1.0 dB) for more configurations
3. **Multiple signal frequencies**: Run the script for different signal frequencies to find optimal pump settings across your bandwidth
4. **CSV analysis**: Export to CSV and use tools like Excel or Python for custom analysis and visualization

## Integration with twpa_noise_measurement

The generated YAML can be directly merged into your measurement configuration. See `CUSTOM_LIST_MODE.md` for details on the custom list mode.

Example complete configuration:

```yaml
# Your existing configuration
general:
  base_path: /path/to/data
  # ...

measurement:
  name: equal_gain_noise_test
  device: RANV502D1
  device_type: single_twpa
  # ...

# Paste the generated sweep section here
sweep:
  mode: custom_list
  loop_order:
    - temperature
  axes:
    temperature: [0.2, 0.3, 0.4]
  custom_configs:
    # Generated configurations
    - pump1_power: -8.5
      pump1_frequency: 8.44e9
      pump1_phase: 0.0
    # ...

live_plot:
  enabled: true
```

## Troubleshooting

**No configurations found:**
- Check that the measurements directory path is correct
- Verify the directory structure matches the expected format
- Ensure CSV files contain the required columns

**Gain values seem wrong:**
- Check if you need to provide `--reference-s21` for proper gain calculation
- Verify the signal frequency is within the measured range

**Too few/many configurations:**
- Adjust `--tolerance` to get more or fewer matches
- Try different `--target-gain` values based on the initial scan

## Example Output

```bash
$ python find_equal_gain_configs.py \
    --measurements-dir ./measurements \
    --signal-freq 6.5e9 \
    --target-gain 12.0 \
    --tolerance 0.5 \
    --output-yaml equal_gain.yaml

Scanning measurements in: ./measurements
Signal frequency: 6.500 GHz
Filtering for gain: 12.00 ± 0.50 dB

================================================================================
Found 15 pump configurations at signal frequency 6.500 GHz
================================================================================
Pump Power (dBm)     Pump Freq (GHz)      Gain (dB)      
--------------------------------------------------------------------------------
-8.00                8.462                12.45          
-8.25                8.451                12.38          
-8.50                8.440                12.12          
-8.75                8.429                11.89          
-9.00                8.418                11.67          
-9.25                8.407                12.23          
-9.50                8.396                12.41          
-9.75                8.385                12.18          
-10.00               8.374                11.95          
-10.25               8.363                12.34          
-10.50               8.352                12.29          
-10.75               8.341                12.07          
-11.00               8.330                11.84          
-11.25               8.319                12.15          
-11.50               8.308                12.38          
--------------------------------------------------------------------------------
Gain statistics:
  Mean: 12.16 dB
  Std:  0.24 dB
  Min:  11.67 dB
  Max:  12.45 dB
================================================================================

Generated custom_configs YAML: equal_gain.yaml
Total configurations: 15
Temperature points: 3
Total measurements (with probe freqs): 15 × 3 × N_probe_freqs