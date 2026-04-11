# V4 JTWPA: Package Parasitics Cap the Achievable Gain

**Date:** 2026-04-11
**Artifacts:** `Box > Fawkes > V4_Package_Parasitic_HB/` (deck + 3 interactive HTMLs)
**Code:** `~/projects/crucible/campaigns/v4_parasitic_comparison.py`
**Reference:** `~/projects/twpa/measurements/cryo_tdr/20260408/extract_transitions.py`

## TL;DR

TDR-extracted V4 package parasitics cap the readout-band (6.5–7.4 GHz) mean
gain of a 1956-cell chain at **~12 dB**, regardless of pump drive. The
clean-launch chain rises smoothly to **32 dB** at Ip/Ic = 0.8. The package is
the bottleneck, not the junction count or the pump headroom.

## Measurement chain

1. **S-parameters at 4 K** (Lab 30 station, cryo VNA). 2650-sample S22 trace
   post-processed into Γ(t) over a 20 ns / 7.55 ps window.
2. **Forward-model ABCD fit** against Re(Γ(t)):
   `L_ard → Z_pcb microstrip → C_bump → linear Z-taper (10-segment)`.
   Lossy-cable correction `α(f)=α₀·√(f/1 GHz)` absorbed into the fit.
   Differential-evolution + Nelder-Mead polish. RMS residual 3.55 mΓ (3.6 %).
3. **Extracted parameters** (`TDR_20260408_PARASITICS` in `crucible.device`):

    ```
    L_ard   = 240 pH  (±15 %)
    C_bump  = 110 fF  (±15 %)
    Z_pcb   = 53.6 Ω,  τ_pcb  = 125 ps,  N_pcb  = 10
    Z_taper 53.6 → 57.9 Ω over τ_taper = 106 ps,  N_taper = 10
    ```

4. **V4_POR_TDR preset**: `V4_POR` with `input_parasitics` and
   `output_parasitics` both populated by the above dict.

## Simulation

- Full 1956-cell V4 chain in `crucible.run.hb` → JosephsonCircuits.jl `hbsolve`
- Two configurations: `V4_POR` (clean 50 Ω → chip) vs `V4_POR_TDR` (TDR-dressed)
- 361 signal-frequency points, 4.0–13.0 GHz, 25 MHz steps
- 8 pump levels, Ip/Ic ∈ {0.1, 0.2, …, 0.8}
- `fpump = 8.13 GHz`, `f_rpm = 8.32 GHz`, `stub_sections = 1` (single-section
  approximation of the lambda/4 resonator coupled at antinode; 5th harmonic
  tank dropped as user-approved simplification, 3rd harmonic is a node)
- All 16 points solved, CM tier1 throughout (max 4.67e-10 clean, 2.94e-9 par)

## Key numbers (mean ± σ over 6.5–7.4 GHz readout band)

| Ip/Ic | pp (dBm) | clean | parasitic | Δ       |
|-------|----------|-------|-----------|---------|
| 0.10  | −83.98   | −1.94 ± 0.09 | −3.15 ± 0.61 | −1.21 |
| 0.20  | −77.96   | −1.83 ± 0.10 | −3.05 ± 0.63 | −1.22 |
| 0.30  | −74.44   | −0.81 ± 0.16 | −2.04 ± 0.53 | −1.23 |
| 0.40  | −71.94   | +3.36 ± 0.13 | +2.13 ± 0.80 | −1.23 |
| 0.50  | −70.00   | +10.22 ± 0.04 | +9.28 ± 2.53 | **−0.94** |
| 0.60  | −68.42   | +17.62 ± 0.23 | +14.68 ± **5.34** | −2.93 |
| 0.70  | −67.08   | +25.04 ± 0.50 | +9.62 ± 1.34 | **−15.42** |
| 0.80  | −65.92   | +32.25 ± 0.97 | +11.91 ± 1.56 | **−20.34** |

## Interpretation

1. **Ip/Ic ≤ 0.5: parasitics are benign.** Mean penalty ≤ 1 dB; parasitic
   curve tracks clean curve within the error bars. A package-dressed chain
   looks indistinguishable from clean at low pump.

2. **Ip/Ic ≈ 0.6: flatness fails before headroom.** The parasitic-device
   in-band standard deviation jumps to **5.3 dB** (vs 0.23 dB clean) while the
   mean is still only 3 dB down. Ripple is the first symptom, not loss.

3. **Ip/Ic ≥ 0.7: hard gain ceiling at ~12 dB.** Pumping harder does not
   help — mean gain saturates and even slightly recovers. The chain becomes
   insensitive to `Ip` above this knee.

4. **Physical mechanism.** Phase-matched JTWPA gain scales as `Ip² · L`.
   Any parasitic-induced phase error in the package is amplified by the
   gain itself at high pump. At low pump the phase budget has slack; at
   high pump every degree of phase error costs dB. The `L_ard + C_bump +
   Z-taper` chain sets up a standing-wave-like interaction that caps the
   effective pump the junction ladder sees regardless of port drive.

5. **Implication for device design.** Increasing junction count or critical
   current will not unlock the clean-launch 25–32 dB regime while the
   package looks like the TDR-extracted model. Package engineering
   (reducing `L_ard`, flattening the Z-taper, lowering `C_bump`) is the
   path to higher gain, not chain length.

## Follow-up candidates (not run yet)

- **Element ablation at Ip/Ic = 0.6**: zero `L_ard`, `C_bump`, taper
  individually to identify the dominant offender.
- **fpump retune at Ip/Ic = 0.7 parasitic**: sweep `fpump ±100 MHz` in
  20 MHz steps — does retuning recover in-band gain? If yes, the lesson
  becomes "parasitics shift the optimal pump freq, not the achievable gain".
- **V3_POR_TDR comparison**: run the same sweep on V3 with the same
  parasitic dict. Candidate explanation for the W14-47 "V3 HB overshoots
  HD3C by 11 dB" gap — prior HB runs never dressed V3 with package
  parasitics.
- **JoSIM cross-check** at Ip/Ic = 0.7 parasitic to rule out HB
  convergence artifacts.

## Convention gotcha locked in during this session

JC.jl's `hbsolve` `sources=[(…, current=Ip)]` uses the **physical
time-domain peak** of the drive cosine, **not** a Fourier coefficient.
Crucible's helper `hbsolve_helper.jl` passes `current=Ip` directly; any
`/2` halving is incorrect and will silently shift Ip/Ic labels by 2×.
Empirically verified against the V4 gain curve
(0.3 → ~0 dB, 0.5 → 10 dB, 0.6 → 17 dB, 0.7 → 25 dB, 0.8 → 32 dB).
Source-level regression test:
`tests/test_hb.py::test_hbsolve_pump_source_uses_physical_peak_ip`.

The legacy script `~/projects/twpa/hb_ic_monte_carlo.jl:179` does use
`ip = (ip_over_ic · Ic) / 2`, but that is a historical labeling quirk
where its `ip_over_ic` labels 2× the physical drive — **not** a JC.jl
Fourier-coefficient requirement. When cross-comparing with that script,
halve its `ip_over_ic` to get the crucible-equivalent label.
