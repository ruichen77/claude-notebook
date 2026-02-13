# Quantum Vault

Personal reference for superconducting qubit simulation and DTC (Double Transmon Coupler) research.

---

## Concepts
- [[Quantum Simulation Concepts]] — Core QM, Hamiltonians, bases, operators, eigenstates
- [[DTC State Labeling]] — Energy index vs physical photon count gotcha
- [[Rockbottom Gotchas]] — Critical gotchas for the rockbottom/SmarterThanARock simulator

## Simulation Tools
- [[Spectroscopy Prediction Workflow]] — Full pipeline: simDict → M = V†QV → 2D spectroscopy map
- [[Coupling and Transition Matrix Visualization]] — H_c and Q_qubit matrix elements in dressed basis

## Implementation Plans
- [[Dispersive Coupler Implementation Plan]] — χ calculation with both-branch tracking at avoided crossings

## Benchmarks & Debugging
- [[Parallel Simulation Benchmarks]] — landsman2 core scaling, NUMA effects, timing estimates
- [[Chi Sweep Debugging Notes]] — Flux range issues, performance optimization notes

---

## Quick Links

| System      | Dimensions | Optimal Cores | Time (101 pts) |
| ----------- | ---------- | ------------- | -------------- |
| Q-DTC-Q     | 432        | 28            | ~7 sec         |
| R-Q-DTC-Q-R | 3888       | 21            | ~19 min        |

## Repositories

| Repo | Location | Purpose |
|------|----------|---------|
| EnhancedSmarterThanARock | `landsman2:~/repos/EnhancedSmarterThanARock/` | Core simulation library |
| dtc_spectroscopy_prediction | `landsman2:~/repos/dtc_spectroscopy_prediction/` | Spectroscopy maps |
| coupling_matrix_viz | `landsman2:~/repos/coupling_matrix_viz/` | Interactive matrix visualization |
| dispersive_shift_calculator | `landsman2:~/repos/dispersive_shift_calculator/` | χ calculations |
