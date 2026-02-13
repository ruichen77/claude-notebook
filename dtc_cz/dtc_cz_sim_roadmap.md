# DTC CZ Gate Error Simulator — Roadmap

## Goal
Study how **coupler coherence** (T1 and T_phi of the DTC) limits CZ gate fidelity,
using the existing charge-basis DTC Hamiltonian (validated to machine precision
against EnhancedSmarterThanARock).

## What Exists (Layers 0-1)
- **dtc_hamiltonian.py** — Full charge-basis DTC Hamiltonian, 432-dim Hilbert space
  for Q-DTC-Q (Big Endeavour params). Diagonalizes, labels states, computes ZZ.
- **flux_sweep.py** — ZZ vs flux, physics-aware idle point detection (avoids
  anticrossing artifacts), finds idle phi=0.325 and safe interaction region.
- **pulse.py** — Square, Slepian, cosine, net-zero pulse shapes + sampling.
- **adiabatic_sim.py** — Layer 0: integrates ZZ along pulse trajectory for
  conditional phase. Fast interpolated version for parameter sweeps.
- **unitary_sim.py** — Layer 1: piecewise-constant Schrodinger evolution,
  gate fidelity, leakage, conditional phase extraction.
- **lindblad_sim.py** — Layer 2 skeleton: mesolve with constant T1/T2,
  process fidelity from density matrices.
- **analysis.py** — Plotly/matplotlib plotting with bold black borders.

## What Needs To Be Done

### Step 1: Layer 0 — CZ Pulse Design  [CURRENT]
Find a working (amplitude, T_gate) operating point using adiabatic phase integral.
- Add trapezoid pulse (flat-top with cosine rise/fall) to pulse.py
- Sweep (amplitude, T_gate) space, find theta_CZ = pi contour
- Select operating point, validate with Layer 1

### Step 2: Layer 1 — Coherent Baseline
Run unitary evolution at the operating point.
- Coherent gate fidelity (floor for all subsequent analysis)
- Leakage L1, conditional phase error delta_theta, single-qubit Z phases
- Population dynamics for |11> input (coupler excitation during pulse)

### Step 3: Flux-Dependent Coupler Decoherence Model
**New module: coupler_decoherence.py**

During the CZ pulse, the DTC moves away from its sweet spot. Its coherence
degrades exactly when the gate is happening. Two effects:

- **T1(phi)** — Purcell loss increases as coupler frequency drops toward
  readout resonator. Parameterized model:
  - Dielectric floor: gamma_diel = omega_C / Q_diel
  - Purcell: gamma_purcell = g_r^2 * kappa_r / ((omega_C - omega_r)^2 + (kappa_r/2)^2)
  - Idle baseline: gamma_idle = 1/T1_idle
  - Total: T1(phi) = 1 / (gamma_idle + gamma_purcell + gamma_diel)

- **T_phi(phi)** — 1/f flux noise dephasing grows with |d(omega_C)/d(phi)|.
  At the sweet spot |d(omega)/d(phi)| = 0, so T_phi -> infinity.
  Away from sweet spot: gamma_phi = |d(omega_C)/d(phi)| * A_flux * sqrt(ln(2*pi*f_IR*T_gate))

- **Coupler participation eta_C(phi)** — During the pulse, |11>_dressed picks
  up coupler character near the anticrossing. This amplifies decoherence.

### Step 4: Extend Lindblad Simulation
Extend lindblad_sim.py with:
- **Time-dependent collapse operators** — coupler relaxation and dephasing
  rates change at each time step as phi(t) changes
- **5-configuration error budget**:
  - A: No decoherence (sesolve) -> coherent error floor
  - B: Coupler T1 only -> isolates relaxation
  - C: Coupler T_phi only -> isolates dephasing
  - D: Coupler T1 + T_phi -> all coupler decoherence
  - E: Full (coupler + qubits) -> total gate error
- Error decomposition by subtraction: each channel's contribution

### Step 5: Coupler Coherence Sweeps
**New module: sweep.py**
- 1D sweep: Gate fidelity vs T1_C (1 us to 100 us), fixed T2
- 1D sweep: Gate fidelity vs T2_C (0.5 us to 50 us), fixed T1
- 2D sweep: Fidelity surface on (T1_C, T2_C) grid
- Full error budget at each sweep point (run all 5 configs)
- Parallelized with multiprocessing (each point is independent)

### Step 6: Diagnostic Plots
1. Energy spectrum vs flux (with pulse excursion marked)
2. ZZ vs flux
3. Coupler T1(phi) and T_phi(phi) vs flux
4. Pulse shape + population dynamics during |11> evolution
5. Gate fidelity vs coupler T1
6. Gate fidelity vs coupler T2
7. 2D fidelity surface (T1 x T2 heatmap)
8. Error budget stacked bar chart at baseline
9. Error budget evolution vs coupler T1 (stacked area)

## Key Physics
- The DTC CZ gate works by flux-pulsing the coupler toward an anticrossing
  where ZZ grows large, accumulating a pi conditional phase.
- The gate time is set by how much ZZ is available in the safe region
  (away from state-swap anticrossings).
- Longer gates -> more decoherence -> coupler coherence matters more.
- The coupler's coherence degrades during the pulse (away from sweet spot),
  amplifying the error.
- This creates a speed-fidelity tradeoff: pulse harder -> more ZZ -> faster gate
  -> less decoherence, BUT closer to anticrossing -> more leakage.

## Device Parameters (Big Endeavour)
- Q0: f01=4.5 GHz, anharm=0.21 GHz, cutoff=6
- DT1: f01=3.8 GHz, anharm=0.08 GHz, g=0.7, phi_idle=0.4, cutoff=12
- Q1: f01=5.0 GHz, anharm=0.21 GHz, cutoff=6
- Couplings: Q0-DT1=0.18, Q1-DT1=0.18, DT1_internal=0.01
- Hilbert space: 6 x 12 x 6 = 432 dim

## Known Issues
- Safe-region ZZ is only ~1.5 MHz (max at safe boundary phi=0.2275).
  This means CZ gate time is ~1 us, making coupler coherence very relevant.
- ZZ is negative throughout, so target CZ phase is -pi.
- The anticrossing region (phi=0.09-0.225) must be avoided by the pulse.
