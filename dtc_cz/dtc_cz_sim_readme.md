# dtc_cz_sim

CZ gate error simulator for Double-Transmon Coupler (DTC) systems.

Accepts the same `simDict` format as EnhancedSmarterThanARock for circuit definition,
plus a `czParams` dict for gate-specific parameters.

## Simulation Layers

| Layer | Module | Physics |
|-------|--------|---------|
| 0 | `adiabatic_sim` | Adiabatic phase integral |
| 1 | `unitary_sim` | Time-dependent Schrodinger (non-adiabatic leakage) |
| 2 | `lindblad_sim` | Qubit T1/T2 decoherence |
| 3 | `lindblad_sim` | Coupler flux-dependent decoherence |
| 5 | `flux_noise` | Quasi-static 1/f flux noise |
| 6 | `tls_model` | Explicit TLS defects |

## Installation

```bash
pip install -e .
```

## Quick Start

```python
from dtc_cz_sim import DTCSystem, adiabatic_cz_phase, slepian_pulse

simDict = {
    'transmons': {
        'Q0': {'f01': 4.5, 'anharm': 0.21, 'cutoff': 6},
        'DT1': {'f01': 3.8, 'anharm': 0.08, 'g': 0.7, 'phi': 0.4, 'cutoff': 12},
        'Q1': {'f01': 5.0, 'anharm': 0.21, 'cutoff': 6},
    },
    'couplings': [
        ['Q0', 'DT1;0', 0.18],
        ['Q1', 'DT1;1', 0.18],
        ['DT1;0', 'DT1;1', 0.01],
    ],
    'N': 12,
    'cutoff': 5,
}

system = DTCSystem(simDict)
```
