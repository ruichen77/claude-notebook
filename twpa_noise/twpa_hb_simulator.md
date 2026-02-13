# TWPA Harmonic Balance Simulator

A Julia-based parallel simulation framework for Traveling Wave Parametric Amplifiers (TWPAs) using harmonic balance analysis via the JosephsonCircuits.jl package.

## 🚀 Features

- **Parallel Execution**: Leverages Julia's multi-threading for efficient parameter sweeps
- **Flexible Parameter Sweeps**: Sweep any circuit or operating parameter
- **Pump Power Handling**: Automatic conversion from pump power (dBm) to pump current with impedance calculation
- **Thread-Safe Design**: Each simulation point uses its own circuit instance to avoid race conditions
- **Comprehensive Visualization**: Generates publication-quality plots with Plotly.js
- **Modular Architecture**: Clean separation of concerns for easy maintenance and extension

## 📋 Requirements

- Julia 1.8 or higher
- JosephsonCircuits.jl package
- Additional Julia packages (automatically installed):
  - TOML, Dates, ProgressMeter, Plots, PlotlyJS
  - Symbolics, JLD2, FLoops, ArgParse

## 🛠️ Installation

1. Clone the repository:
```bash
git clone https://github.com/yourusername/TWPA_HB_simulator.git
cd TWPA_HB_simulator
```

2. Install Julia dependencies:
```julia
using Pkg
Pkg.activate(".")
Pkg.instantiate()
```

## 💻 Usage

### Basic Command Line Usage

```bash
# Run with default configuration
julia --threads 8 run_simulation.jl experiment_name

# Sweep a specific parameter
julia --threads 8 run_simulation.jl -s Ic --start 3.0 --stop 5.0 --step 0.2 ic_sweep

# Use a configuration file
julia --threads 8 run_simulation.jl --config my_config.toml my_experiment

# List available parameters
julia run_simulation.jl --list-params
```

### Command Line Options

| Option | Description | Example |
|--------|-------------|---------|
| `-s, --sweep-param` | Parameter to sweep | `Ic`, `Cg`, `pump_power_dBm` |
| `--start` | Start value for sweep | `3.0` |
| `--stop` | Stop value for sweep | `5.0` |
| `--step` | Step size for sweep | `0.2` |
| `-c, --config` | Configuration file path | `config.toml` |
| `-t, --threads` | Number of threads | `8` |
| `-o, --output-dir` | Output directory | `./results` |
| `--list-params` | List available parameters | - |
| `--dry-run` | Validate config without running | - |
| `--quiet` | Suppress output | - |
| `--debug` | Enable debug output | - |

### Available Sweep Parameters

| Parameter | Unit | Description |
|-----------|------|-------------|
| `Ic` | µA | Critical current |
| `Ip` | µA | Pump current (direct) |
| `pump_power_dBm` | dBm | Pump power (converts to current) |
| `Cg` | fF | Ground capacitance |
| `Cc` / `Ccs` | aF | Coupling capacitance |
| `Cj` | fF | Junction capacitance |
| `L_disp` | pH | Dispersion inductance |
| `pump_freq` | GHz | Pump frequency |
| `Q_TWPA` | - | TWPA quality factor |

### Configuration File (TOML)

Create a `config.toml` file to define simulation parameters:

```toml
[experiment]
name = "twpa_pump_sweep"
output_dir = "./results"

[circuit]
num_junctions = 1956
pmr_pitch = 14

[circuit.quality_factors]
TWPA_mode = 360.0
JJ = 300.0
TL = 833.0

[circuit.base_parameters]
Ic = 4.25e-6          # Critical current (A)
L_disp = 350e-12      # Dispersion inductance (H)
Cg = 20e-15           # Ground capacitance (F)
Cc = 14e-15           # Coupling capacitance (F)
Cj = 37e-15           # Junction capacitance (F)
R_termination = 50.0  # Termination resistance (Ω)

[sweep]
parameter = "pump_power_dBm"
start = 6.0
stop = 10.0
step = 0.2

# Optional: Custom description and unit
description = "Pump Power at Device"
unit = "dBm"

# Pump power configuration (for pump_power_dBm sweeps)
[sweep.pump_power_config]
impedance_mode = "calculated"  # "calculated", "fixed", or "auto"
coupling_efficiency = 0.98     # Directional coupler efficiency
line_attenuation_dB = 73.0     # Total line attenuation
power_reference = "source"      # "source" or "device"

[simulation]
frequency_range = [4.0, 14.0, 0.01]  # [start, stop, step] in GHz
pump_frequency = 8.135                # GHz
pump_harmonics = 20
modulation_harmonics = 10

[visualization]
generate_plots = true
formats = ["png", "html"]
dpi = 300
color_scheme = "viridis"
save_data_matrix = true
generate_interactive = true
plotly_theme = "plotly_white"
show_pump_power = true
marker_shape = "circle"  # "circle", "square", "diamond", "none"

[parallel]
enabled = true
num_threads = 8
```

## 📁 Project Structure

```
TWPA_HB_simulator/
├── src/
│   ├── JosephsonSimulator.jl    # Main module
│   ├── SharedTypes.jl           # Type definitions
│   ├── Config.jl                # Configuration management
│   ├── CLI.jl                   # Command-line interface
│   ├── CircuitBuilder.jl        # Circuit construction
│   ├── Simulation.jl            # Simulation engine
│   ├── SimulationMonitor.jl     # Progress tracking
│   ├── PumpPowerConversion.jl   # Power-to-current conversion
│   ├── PostProcessing.jl        # Results analysis
│   ├── Visualization.jl         # Plotting functions
│   ├── Utils.jl                 # Utility functions
│   └── ContextualLogger.jl      # Logging utilities
├── run_simulation.jl            # Main entry point
├── configs/                     # Example configurations
│   └── example_config.toml
├── results/                     # Output directory (auto-created)
└── README.md                    # This file
```

## 🏗️ Architecture

### Module Hierarchy

```
JosephsonSimulator (Main Module)
├── SharedTypes         # Shared data structures
├── Config             # Configuration loading and validation
├── CLI                # Command-line argument parsing
├── CircuitBuilder     # TWPA circuit construction
├── Simulation         # Core simulation engine
├── PumpPowerConversion # Power/current conversions
├── SimulationMonitor  # Progress tracking
├── PostProcessing     # Data analysis
├── Visualization      # Plotting and visualization
└── Utils              # Helper functions
```

### Key Design Features

1. **Thread-Safe Parallel Execution**: Each thread receives its own pre-built circuit to avoid symbolic variable conflicts
2. **Modular Design**: Clear separation between configuration, simulation, and visualization
3. **Flexible Parameter Sweeps**: Any circuit parameter can be swept with proper impedance recalculation
4. **Smart Pump Power Handling**: Automatic impedance calculation and power-to-current conversion

### Execution Flow

1. **Configuration Phase** (Single-threaded)
   - Load TOML configuration or parse command-line arguments
   - Create simulation configuration structure
   - Validate parameters

2. **Circuit Pre-building** (Single-threaded)
   - Create individual circuits for each sweep point
   - Apply swept parameter to circuit structure
   - Avoid symbolic variable conflicts

3. **Parallel Simulation** (Multi-threaded)
   - Each thread processes assigned sweep points
   - Uses pre-built circuit for its simulation
   - Thread-safe progress tracking

4. **Post-processing** (Single-threaded)
   - Collect results from all threads
   - Generate plots and analysis
   - Save results to disk

## 🔧 Developer Notes

### Adding New Sweep Parameters

1. Add parameter definition in `Config.jl`:
```julia
function get_sweep_parameters(param_name::String)
    sweep_configs = Dict(
        # ... existing parameters ...
        "new_param" => (
            unit_scale=1e-9,
            description="New Parameter",
            unit="nH",
            default_start=1.0,
            default_stop=10.0,
            default_step=0.5
        )
    )
end
```

2. Handle the parameter in `create_swept_circuit_config()` (Simulation.jl):
```julia
elseif param_name == "new_param"
    modified_params["new_param"] = param_value
```

3. If needed, handle in `apply_sweep_parameter()` (CircuitBuilder.jl) for source-related changes

### Debugging Thread Issues

If you encounter segmentation faults:

1. **Try fewer threads**:
```bash
julia --threads 2 run_simulation.jl experiment_name
```

2. **Run in serial mode**:
```toml
[parallel]
enabled = false
```

3. **Clear precompiled cache**:
```bash
rm -rf ~/.julia/compiled/v1.*/JosephsonSimulator
```

### Performance Optimization

- **Thread Count**: Use number of physical cores (not hyperthreads) for best performance
- **Memory**: Each simulation uses ~100-500 MB depending on frequency points and harmonics
- **Batch Size**: For very large sweeps (>100 points), consider breaking into batches

## 📊 Output Files

The simulator generates several output files in the results directory:

```
results/
└── YYYY-MM-DD_HHMMSS_experiment_name_param/
    ├── config.toml              # Copy of configuration used
    ├── results.jld2             # Raw simulation data
    ├── S21_overlay.png/.html    # S-parameter plots
    ├── heatmap.png/.html        # 2D frequency-parameter plot
    ├── gain_vs_param.png/.html  # Gain variation plot
    └── individual_plots/        # Per sweep point plots
        ├── S21_param_value1.png
        └── ...
```

## 🐛 Troubleshooting

### Common Issues

1. **"Index out of bounds for pump power array"**
   - Solution: Ensure your sweep range matches the configuration
   - This should be fixed with the current implementation

2. **Segmentation fault during parallel execution**
   - Solution: Reduce thread count or run in serial mode
   - The Option B implementation should prevent this

3. **"Unknown sweep parameter"**
   - Solution: Check available parameters with `--list-params`
   - Ensure parameter name matches exactly (case-sensitive)

4. **Memory errors**
   - Solution: Reduce frequency points or harmonic counts
   - Use fewer threads to reduce memory overhead

## 📝 Citation

If you use this simulator in your research, please cite:
- JosephsonCircuits.jl package
- Your publication/repository

## 📄 License

[Your License Here]

## 🤝 Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## 📧 Contact

[Your contact information]

## 🙏 Acknowledgments

- JosephsonCircuits.jl developers
- Julia community
- [Other acknowledgments]