# CZ Gate Pulse Design Intuition — DTC Architecture

## Core Gate Mechanism

The CZ gate works by flux-pulsing the coupler to bring |11⟩ near the |20⟩ (or |02⟩) avoided crossing. Population makes a round trip — |11⟩ → |20⟩ → |11⟩ — accumulating a π conditional phase. The gate succeeds when the population returns cleanly to |11⟩ with exactly π phase.

## Computing the CZ Unitary

1. **Build H(t):** Static Hamiltonian from circuit quantization (scqubits) + time-dependent flux pulse on coupler.
2. **Time-evolve:** Propagate U = T·exp(−i∫H(t)dt) via sequential matrix exponentiation or ODE solve (QuTiP `sesolve`/`propagator`).
3. **Project:** Extract 4×4 computational subspace block from full propagator.
4. **Extract CZ phase:** φ_ZZ = arg(U₁₁) − arg(U₁₀) − arg(U₀₁) + arg(U₀₀). Target: π.

## Lab Frame vs Rotating Frame

**Problem:** Lab-frame simulations track GHz-scale free precession that is irrelevant to the gate. This causes:

- Fast population oscillations that obscure slow ZZ dynamics
- ODE solver phase accumulation errors over thousands of cycles
- Fidelity hypersensitivity to sub-ns gate timing
- Numerical precision loss when subtracting large single-qubit phases to extract the small conditional phase

**Fix:** Transform into interaction picture w.r.t. H₀ (diagonal/bare frequencies). New Hamiltonian only contains the slow interaction terms. Populations are frame-independent, but solver accuracy improves dramatically.

**Key distinction:**
- ZZ phase integral (∫ζ(t)dt) is immune — it's already a differential quantity where bare frequencies cancel.
- Full unitary fidelity calculation is severely impacted — rotating frame is essential for trustworthy numbers.

## State Labeling in the Dressed Basis

At idle flux, eigenstates are mostly bare product states — label by maximum overlap. Track labels adiabatically as flux tunes via overlap continuity between adjacent flux points (scqubits `ParameterSweep` does this natively). Labels become ambiguous at avoided crossings by definition — but you only need clean labels at the start and end of the pulse (idle flux), not during it.

## Error Taxonomy

### Leakage (L₁)
Population starts in computational subspace, ends outside (e.g., stuck in |20⟩). Detectable by tracking computational subspace population / state purity.

### Seepage (L₂)
Population excurses to |20⟩, loses an excitation (e.g., coupler T₁ decay), and falls back into the computational subspace — but to the **wrong** state (|10⟩, |01⟩, or |00⟩ instead of |11⟩). **Invisible to purity tracking.** Appears as a bit-flip error indistinguishable from gate error in standard RB.

**Detection:** Wood & Gambetta LRB protocol (Phys. Rev. A 97, 032306, 2018). Run RB starting from both a computational state AND a leakage state. Track computational subspace population in both cases:
- Decay from computational start → L₁ (leakage rate)
- Growth from leakage start → L₂ (seepage rate)
- Steady-state: p_steady = L₁ / (L₁ + L₂)

### Seepage risk scales with time spent in |20⟩
More Rabi oscillation cycles near the avoided crossing = more time exposed to coupler T₁ = higher seepage probability. This is why smoother pulses with fewer oscillations outperform.

## Two Types of Avoided Crossings — Opposite Strategies

### Intentional: |11⟩ ↔ |20⟩
- **Strategy: SLOW / adiabatic**
- Park near the crossing to accumulate conditional phase
- This is the gate mechanism

### Parasitic: TLS defects, spurious coupler modes
- **Strategy: FAST / diabatic**
- Sweep through as rapidly as possible
- Landau-Zener: P_loss = 1 − exp(−2πg²/ℏv), where v = dE/dt
- Fast sweep (large v) → P_loss → 0 (system doesn't notice the crossing)
- Slow sweep (small v) → P_loss → 1 (population transfers to parasitic mode)

### Pulse design tension
Ramp segments must be fast enough to blast through parasitic crossings but slow enough near the target crossing to accumulate phase. Trapezoidal ramp time directly controls this tradeoff.

## Pulse Shape Performance

| Pulse shape | Fidelity (1000 ns) | Notes |
|---|---|---|
| Trapezoid (20 ns ramp) | 0.944 | Fast ramps → less parasitic exposure |
| Trapezoid (40 ns ramp) | 0.924 | Slower ramps → more TLS vulnerability |
| Cosine arch | 0.920 | Smooth but slow through intermediate flux |
| Square (2 ns ramp) | 0.879 | Sharp transient → high bandwidth excitation |
| Slepian | 0.784 | Smooth ramps spend most time at intermediate flux |

Note: These fidelities likely include lab-frame numerical artifacts. Rerun in rotating frame for accurate comparison.

## TLS Defect Considerations

- TLS defects are spatially random and temporally unstable (drift on hours–days timescale)
- Pulsed coherence measurements under gate-like conditions predict performance far better than idle T1/T2
- Pulse optimization must be continuously recalibrated (IBM uses JAZZ + Slepian + RL pipeline)
- TLS crossings along the flux excursion path are probed by the pulse ramps — this is where most parasitic loss occurs

## Key References

- Wood & Gambetta, "Quantification and characterization of leakage errors," Phys. Rev. A 97, 032306 (2018) — LRB protocol, L₁/L₂ framework
- Li et al. — DTC implementation, coupler-dominated error budgets
- Kubo & Goto — Theoretical foundations for DTC CZ gates
- Motzoi, Gambetta et al., Phys. Rev. Lett. 103, 110501 (2009) — DRAG pulses for leakage suppression
