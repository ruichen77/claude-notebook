# duffing_cz Package Reference

Duffing oscillator Q-DTC-Q CZ gate simulation with qiskit-dynamics + JAX pmap.

**Local repo**: `~/repos/dtc_cz_sim_qiskit/`
**Remote (run sims here)**: `landsman3:/home/US8J4928/repos/dtc_cz_sim_qiskit/` (qiskit_dyn conda env)
**Also deployed**: `landsman2:/home/US8J4928/repos/dtc_cz_sim_qiskit/` (no qiskit_dyn env yet)

## Development Workflow

1. **Develop locally** in `~/repos/dtc_cz_sim_qiskit/`
2. **Deploy to remote** for testing:
   ```bash
   scp ~/repos/dtc_cz_sim_qiskit/duffing_cz/*.py landsman3:/home/US8J4928/repos/dtc_cz_sim_qiskit/duffing_cz/
   scp ~/repos/dtc_cz_sim_qiskit/examples/*.py landsman3:/home/US8J4928/repos/dtc_cz_sim_qiskit/examples/
   ```
3. **Run on landsman3**: `conda run -n qiskit_dyn python -u examples/run_amplitude_sweep.py`
4. **Commit on landsman3** after successful test
5. **Copy results locally** to `~/projects/dtc/sim_outputs/`

## Package Structure

```
duffing_cz/
├── __init__.py      # Lazy imports (JAX not loaded until configure_jax called)
├── jax_setup.py     # configure_jax(num_cores, enable_x64, platform)
├── math_utils.py    # hc, freq, cosm, sinm, duffing, duffing_from_transmon, exchange
├── system.py        # DuffingSystemParams dataclass + DuffingSystem class
├── pulse.py         # RamenParams, ramen(), square_ramen(), build_signals()
├── metrics.py       # GateMetrics dataclass + fidelity/leakage/error functions
├── simulation.py    # SolverConfig, SweepConfig, SweepResult, run_sweep(), run_dynamics()
├── plotting.py      # Plotly: population dynamics, sweep metrics, pulse shape
└── io.py            # print_results(), save_results() (backward-compat JSON)
```

## Key Classes

### DuffingSystemParams
All system parameters in MHz with defaults matching Moein's notebook:
- Qubits: fa=5000, fb=5400, Al=-240 MHz
- Coupler: Ejcl=22456, Ejcr=22484, Eccl=Eccr=100 MHz, rm=0.25
- Coupling: g=100 MHz (1% asymmetry)
- Hilbert space: Nq=3 per qubit, Nc=5 per coupler half → 225 dim

### DuffingSystem(params)
Builds operators and Hamiltonians. Key methods:
- `build_H0(phi_ext)` → static Hamiltonian
- `build_H_int_operators()` → [H_cos, H_sin] interaction operators
- `get_idle_dressed_states()` → (Hds, Uds, psi_comp) at idle flux
- `compute_zz(Hds)` → ZZ coupling in angular freq

### RamenParams
Ramen pulse shape parameters (dimensionless):
- det0=1.0, acceleration=2.0, delta=10.0, target=0.5

### SweepConfig
Parameter sweep configuration with classmethods:
- `.amplitude_sweep(gate_time_us, A_start_pi, A_end_pi, n_points)`
- `.gate_time_sweep(amplitude_rad, T_start_us, T_end_us, n_points)`

### SolverConfig
ODE solver: method='jax_odeint', atol=rtol=1e-9, hmax=0.001

## Pulse Shapes

### `ramen(t, tfinal, A, params)` — Full Ramen arch
Crossing-aware adiabatic pulse. Goes 0 → A → 0 following arctan sweep rate profile.

### `square_ramen(t, tfinal, A, ramen_width, params)` — Ramen rise/fall + flat top
- First half of Ramen arch for rise, flat top at A, second half for fall
- `ramen_width` = total rise+fall duration (analogous to 2 × rise_time)
- More commonly used in experiment than full Ramen

### Positive-only protection (both shapes)
Both `ramen()` and `square_ramen()` always compute the Ramen math with |A|,
then flip the sign at the end for negative A. This avoids arctan/tan math
issues when the target goes negative. Same fix as dtc_cz_sim's ramen_pulse.

## Usage Examples

### Amplitude sweep (reproduce original script)
```python
from duffing_cz import configure_jax
num_cores = configure_jax(num_cores=28)

from duffing_cz import (
    DuffingSystem, DuffingSystemParams,
    SweepConfig, RamenParams, run_sweep, print_results, save_results,
)

system = DuffingSystem(DuffingSystemParams())
sweep = SweepConfig.amplitude_sweep(gate_time_us=0.075, A_start_pi=0.35, A_end_pi=0.65, n_points=50)
result = run_sweep(system, sweep, ramen_params=RamenParams(), num_cores=num_cores)
print_results(result)
save_results(result, 'duffing_cz_results.json')
```

### Square Ramen sweep
```python
result = run_sweep(
    system, sweep,
    ramen_params=RamenParams(),
    pulse_type='square_ramen',
    ramen_width=0.020,  # 20 ns rise+fall, in microseconds
    num_cores=num_cores,
)
```

### Single-point dynamics
```python
from duffing_cz import run_dynamics, plot_population_dynamics
dyn = run_dynamics(system, gate_time_us=0.075, amplitude_rad=0.5*np.pi,
                   ramen_params=RamenParams(), n_times=500)
fig = plot_population_dynamics(dyn)
```

## Differences from dtc_cz_sim (RockBottom-based)

| Feature | duffing_cz (this) | dtc_cz_sim |
|---------|-------------------|------------|
| Hamiltonian | Duffing oscillator approximation | Exact transmon (RockBottom) |
| Hilbert space | 225 dim (Nq=3, Nc=5) | 432 dim (cutoffs 6,12,6) |
| ODE solver | qiskit-dynamics jax_odeint | scipy DOP853 or piecewise |
| Parallelism | JAX pmap (28 cores) | multiprocessing |
| Pulse params | Abstract (A in radians) | Physical (phi_idle, phi_interaction) |

## JSON Output Schema (backward-compatible)
```json
{
  "system": {"wa_MHz", "wb_MHz", "Ala_MHz", "Alb_MHz", "g_MHz", "rm", "Nq", "Nc", "size"},
  "sweep": {"Tf_us": [...], "Acal_over_pi": [...]},
  "metrics": {"avg_coh_error": [[...]], "under_rot_error", "avg_leakage", "per_state_leakage"},
  "best": {"A_over_pi", "coh_error", "avg_leakage", "fidelity"},
  "wall_time_s": float
}
```
