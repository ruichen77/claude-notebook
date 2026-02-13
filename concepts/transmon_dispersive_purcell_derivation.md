# Dispersive Shift and Purcell Decay in Transmon-Resonator Systems

## System Overview

We consider a transmon qubit capacitively coupled to a microwave resonator - the fundamental building block of circuit QED. This derivation will show how the qubit-resonator interaction leads to:
1. **Dispersive shift**: State-dependent frequency shift enabling qubit readout
2. **Purcell decay**: Enhanced spontaneous emission through the resonator

---

## 1. System Hamiltonian

### 1.1 Bare Hamiltonian

The uncoupled system consists of:

**Resonator** (harmonic oscillator):
```
Ĥᵣ = ℏωᵣ(â†â + 1/2)
```

**Transmon** (weakly anharmonic oscillator, truncated to 3 levels):
```
Ĥq = ℏωq|1⟩⟨1| + ℏ(2ωq + α)|2⟩⟨2| + ...
```

where:
- `ωᵣ` = resonator frequency
- `ωq` = qubit transition frequency (|0⟩ → |1⟩)
- `α < 0` = anharmonicity (typically α/2π ≈ -200 to -300 MHz)
- `â†, â` = resonator creation/annihilation operators

### 1.2 Interaction Hamiltonian

The capacitive coupling gives a **Jaynes-Cummings interaction**:

```
Ĥᵢₙₜ = ℏg(â†σ̂₋ + âσ̂₊)
```

where:
- `g` = coupling strength (typically g/2π ≈ 50-200 MHz)
- `σ̂₊ = |1⟩⟨0|`, `σ̂₋ = |0⟩⟨1|` = qubit raising/lowering operators

**Full Hamiltonian:**
```
Ĥ = ℏωᵣâ†â + ℏωqσ̂₊σ̂₋ + (ℏα/2)σ̂₊σ̂₊σ̂₋σ̂₋ + ℏg(â†σ̂₋ + âσ̂₊)
```

### 1.3 Dispersive Regime

The **dispersive regime** requires:
```
|Δ| ≡ |ωq - ωᵣ| ≫ g
```

This is the typical operating regime for transmon readout, with detuning Δ/2π ≈ 1-2 GHz.

---

## 2. Dispersive Shift Derivation

### 2.1 Schrieffer-Wolff Transformation

We use a **unitary transformation** to eliminate the coupling term to second order in g/Δ.

**Generator:**
```
Ŝ = (g/Δ)(â†σ̂₋ - âσ̂₊)
```

**Transformed Hamiltonian:**
```
Ĥ' = e^Ŝ Ĥ e^(-Ŝ) ≈ Ĥ + [Ŝ, Ĥ] + (1/2)[Ŝ, [Ŝ, Ĥ]] + ...
```

### 2.2 Second-Order Effective Hamiltonian

After careful calculation (keeping terms to O(g²/Δ)):

```
Ĥₑff = ℏω'ᵣâ†â + ℏω'qσ̂₊σ̂₋ + ℏχâ†âσ̂z + (ℏα/2)σ̂₊σ̂₊σ̂₋σ̂₋
```

where `σ̂z = |1⟩⟨1| - |0⟩⟨0|`.

### 2.3 Dispersive Shift Formula

The **dispersive shift** χ is:

```
χ = g²/Δ · α/(Δ + α)
```

**Simplified form** (when |α| ≪ |Δ|):
```
χ ≈ g²α/Δ²
```

**Physical interpretation:**
- The resonator frequency depends on qubit state: `ωᵣ(|0⟩) = ω'ᵣ - χ`, `ωᵣ(|1⟩) = ω'ᵣ + χ`
- The qubit frequency depends on photon number: `ωq(n) = ω'q + 2nχ`
- This **ac Stark shift** enables dispersive readout

### 2.4 Typical Values

For a typical transmon system:
- `g/2π = 100 MHz`
- `Δ/2π = 1 GHz`
- `α/2π = -250 MHz`

```
χ/2π ≈ (100 MHz)² × (-250 MHz) / (1 GHz)²
χ/2π ≈ -2.5 MHz
```

This gives a **state-dependent frequency shift** of 2χ/2π ≈ 5 MHz, easily resolvable.

---

## 3. Purcell Decay Rate

### 3.1 Physical Picture

The resonator acts as a **decay channel** for the qubit:
- Qubit spontaneously emits into resonator
- Photon leaks out through resonator's coupling to transmission line
- This enhances the qubit's natural decay rate

### 3.2 Fermi's Golden Rule Approach

The Purcell decay rate is derived using **Fermi's Golden Rule** with the resonator as an intermediate state.

**Bare qubit decay rate** (into free space):
```
Γ₀ = 1/T₁⁰
```

**Enhanced decay rate** (Purcell effect):
```
Γₚ = κ · (g/Δ)²
```

where:
- `κ = ωᵣ/Q` = resonator decay rate (photon loss rate)
- `Q` = resonator quality factor

### 3.3 Total T₁ Time

The **total qubit relaxation rate** is:
```
1/T₁ = Γ₀ + Γₚ = 1/T₁⁰ + κ(g/Δ)²
```

**Purcell factor:**
```
Fₚ = Γₚ/Γ₀ = (κ/Γ₀) · (g/Δ)²
```

### 3.4 Purcell Filter Design

To suppress Purcell decay while maintaining dispersive readout:

**Strategy 1: Large detuning**
- Increase |Δ| → reduces Γₚ ∝ 1/Δ²
- But also reduces χ ∝ 1/Δ²
- Trade-off between readout fidelity and T₁

**Strategy 2: Purcell filter**
- Insert bandstop filter between resonator and transmission line
- Filter blocks qubit frequency, passes resonator frequency
- Effective κ → κₑff(ω) that is small at ωq

**Optimal design:**
```
κₑff(ωq) ≪ κₑff(ωᵣ)
```

This allows:
- Fast readout (large κ at ωᵣ)
- Long T₁ (small κₑff at ωq)

### 3.5 Typical Values

For a system with:
- `g/2π = 100 MHz`
- `Δ/2π = 1 GHz`
- `κ/2π = 1 MHz` (Q ≈ 10,000)

```
Γₚ = 2π × 1 MHz × (100 MHz / 1 GHz)²
Γₚ = 2π × 10 kHz
T₁ᴾᵘʳᶜᵉˡˡ = 1/Γₚ ≈ 16 μs
```

Without Purcell filter, this limits T₁. With proper filtering, T₁ > 100 μs is achievable.

---

## 4. Physical Interpretation and Experimental Implications

### 4.1 Dispersive Readout

**Measurement protocol:**
1. Send microwave pulse at frequency ωᵣ into resonator
2. Transmitted/reflected signal has phase shift depending on qubit state
3. Phase difference: `Δφ ≈ 2χ/κ` (for weak measurement)

**Readout fidelity** depends on:
- Signal-to-noise ratio ∝ χ²/κ
- Measurement time vs T₁
- Quantum efficiency of amplifier chain

### 4.2 Qubit-Photon Dressed States

In the dispersive regime, the eigenstates are approximately:
```
|n, g⟩ ≈ |n⟩|g⟩ + (g/Δ)|n+1⟩|e⟩
|n, e⟩ ≈ |n⟩|e⟩ - (g/Δ)|n-1⟩|g⟩
```

Small admixture of opposite states → **ac Stark shift**.

### 4.3 Multi-Level Effects

For real transmons, must consider |2⟩ state:
```
χ₀₁ = g²α/Δ²
χ₁₂ = g²α/(Δ+α)²
```

The **ratio** χ₁₂/χ₀₁ ≠ 1 causes:
- State-dependent dispersive shift
- Readout-induced transitions to |2⟩
- Limits measurement fidelity

### 4.4 Design Trade-offs

**Coupling strength g:**
- Larger g → larger χ → better readout SNR
- But larger g → larger Purcell decay
- Optimal: g/2π ≈ 50-150 MHz

**Detuning Δ:**
- Larger |Δ| → smaller Purcell decay
- But smaller χ → worse readout
- Optimal: Δ/2π ≈ 1-2 GHz

**Resonator decay κ:**
- Larger κ → faster readout
- But larger Purcell decay
- Solution: Purcell filter to make κ frequency-dependent

---

## 5. Summary of Key Formulas

### Dispersive Shift
```
χ = g²α/Δ² (for |α| ≪ |Δ|)

Full: χ = g²/Δ · α/(Δ + α)
```

### Purcell Decay Rate
```
Γₚ = κ(g/Δ)²

T₁ᴾᵘʳᶜᵉˡˡ = Δ²/(κg²)
```

### Readout SNR (simplified)
```
SNR ∝ χ²n̄/κ

where n̄ = average photon number in resonator
```

### Optimal Operating Point
```
Dispersive regime: |Δ| ≫ g
Weak anharmonicity: |α| ≪ |Δ|
Purcell suppression: κₑff(ωq) ≪ κₑff(ωᵣ)
```

---

## 6. Advanced Topics (Brief Overview)

### 6.1 Quantum Non-Demolition (QND) Measurement
- Dispersive readout is QND: measures σ̂z without causing σ̂x, σ̂y transitions
- Requires χ ≫ κ (resolved sideband regime)

### 6.2 Measurement-Induced Dephasing
- Photon shot noise → qubit dephasing
- Dephasing rate: `Γφ ≈ 4χ²n̄/κ`

### 6.3 Straddling Regime
- When Δ ≈ α, neither dispersive nor resonant
- Complex dynamics, avoided in practice

### 6.4 Multi-Qubit Systems
- Cross-Kerr coupling: `Ĥ = ℏχᵢⱼσ̂zⁱσ̂zʲ`
- Enables two-qubit gates and ZZ crosstalk

---

## References and Further Reading

**Foundational Papers:**
1. Blais et al., "Cavity quantum electrodynamics for superconducting electrical circuits" (2004)
2. Schuster et al., "Resolving photon number states in a superconducting circuit" (2007)
3. Reed et al., "High-fidelity readout in circuit quantum electrodynamics using the Jaynes-Cummings nonlinearity" (2010)

**Reviews:**
1. Blais et al., "Circuit quantum electrodynamics" Rev. Mod. Phys. (2021)
2. Krantz et al., "A quantum engineer's guide to superconducting qubits" (2019)

**Purcell Filters:**
1. Jeffrey et al., "Fast accurate state measurement with superconducting qubits" (2014)
2. Bronn et al., "High speed readout of superconducting qubits using a Purcell filter" (2015)

---

## Appendix: Detailed Schrieffer-Wolff Calculation

### Step 1: Commutators

```
[Ŝ, Ĥ₀] = (g/Δ)[â†σ̂₋ - âσ̂₊, ℏωᵣâ†â + ℏωqσ̂₊σ̂₋]
        = ℏg(â†σ̂₋ + âσ̂₊)
        = -Ĥᵢₙₜ
```

This cancels the first-order coupling!

### Step 2: Second-Order Terms

```
[Ŝ, Ĥᵢₙₜ] = (g²/Δ)[â†σ̂₋ - âσ̂₊, â†σ̂₋ + âσ̂₊]
           = (g²/Δ)(â†âσ̂₊σ̂₋ - ââ†σ̂₋σ̂₊)
           = (g²/Δ)(â†â(σ̂₊σ̂₋ - σ̂₋σ̂₊) - σ̂₋σ̂₊)
           = (g²/Δ)(â†âσ̂z - σ̂₋σ̂₊)
```

Including anharmonicity corrections gives the full dispersive shift formula.

---

*This derivation provides the foundation for understanding transmon-based quantum processors and designing high-fidelity qubit readout schemes.*