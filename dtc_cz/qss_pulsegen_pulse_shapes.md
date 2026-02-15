# qss-pulsegen Pulse Shape Reference

**Repo**: `landsman2:/home/US8J4928/repos/qss-pulsegen/`
**Source**: `src/qss_pulsegen/pulse_shapes.py`

## Pulse Shape Catalog

### Basic Envelopes
- `square(width, amp)` — constant amplitude
- `delay(width)` — zero amplitude
- `cosine(width, amp)` — (1+cos)/2 arch
- `sine(width, amp)` — single sine period
- `sawtooth(width, amp)` — linear ramp to amp

### Gaussian Family
- `gaussian(width, amp, sigma)` — truncated Gaussian, baseline-subtracted
- `gaussian_square(width, amp, sigma, risefall)` — Gaussian rise/fall + flat top (the standard IBM single-qubit gate pulse)
- `gaussian_square_ice(...)` — compound (chunked) version for AWG memory efficiency
- `gaussian_square_half(...)` — Gaussian rise + abrupt end
- `chopped_gaussian_square(...)` — truncated variant for flux pulses
- `gaussian_square_echo(...)` — Gaussian square with ± echo for CR gates

### DRAG Family (leakage suppression via derivative quadrature)
- `drag(width, amp, delta, sigma)` — Gaussian + i·delta·dG/dt on Q-quadrature
- `cdrag(width, amp, delta, sigma, framechange)` — DRAG with chirp/frame rotation
- `cosine_drag(width, amp, delta)` — cosine + derivative DRAG
- `gaussian_square_drag(...)` — Gaussian square with DRAG on rise/fall
- `drag_modu(width, amp, amp1, delta, f, sampling_rate, sigma)` — DRAG with sinusoidal modulation for enhanced leakage suppression

### Ramen Family (crossing-aware adiabatic pulses)
- **`ramen_pulse(width, target, det0, acceleration, delta)`** — core Ramen shape
- `top_ramen(width, amp, target, det0, acceleration, delta)` — unit-height Ramen scaled by amp
- **`square_ramen_pulse(width, ramen_width, target, det0, acceleration, delta)`** — Ramen rise/fall + square flat top (compound pulse)
- `top_ramen_modu(...)` — Ramen with sinusoidal modulation for leakage suppression

### Trapezoid
- `trapezoid(width, amp, sigma)` — linear rise/fall + flat top (compound pulse)
- `reset(width, amp0, amp1, sigma, sigma1)` — trapezoid between two non-zero levels

### Specialized
- `RIP_Spline(width, amp)` — 7th-order polynomial (Cross & Gambetta, PRA 91, 032325)
- `boxcar(width, amp, bandwidth)` — square + FIR low-pass filter
- `spline(width, amp, t, c, k)` — B-spline from knots
- `relay_qj(...)` — qubit-cable state relay (coupling + detuning coordinated pulses)
- `sample_pulse(width, samples)` — arbitrary waveform from sample list
- `custom(width, filename, channel, pulse_name)` — load from JSON file

---

## The Ramen Pulse — Detailed Analysis

### Mathematical Form

Time variable: t in [-pi, pi] mapped to pulse duration.

```
f(t) = det0 + delta * tan( [A * g(t) + B * h(t)] / D )
```

where:
- A = arctan((det0 - target) / delta)
- B = arctan(det0 / delta)
- g(t) = -2a*pi*arctan(a*pi) + 2a*t*arctan(a*t) + log(1+a^2*pi^2) - log(1+a^2*t^2)
- h(t) = -2a*t*arctan(a*t) + log(1+a^2*t^2)
- D = 2a*pi*arctan(a*pi) - log(1+a^2*pi^2)
- a = acceleration parameter
- delta = avoided crossing gap

### Boundary Values
- f(-pi) = f(+pi) = 0  (starts and ends at zero)
- f(0) = target  (reaches target at midpoint)

### Parameters
| Parameter | Meaning |
|-----------|---------|
| `target` | Pulse amplitude at midpoint (the flux excursion target) |
| `det0` | Initial detuning of the TLS from the avoided crossing |
| `delta` | Avoided crossing gap size |
| `acceleration` | Controls sweep rate profile; higher = faster angular velocity |

### Physical Principle

For a two-level system H = (epsilon/2)*sigma_z + (Delta/2)*sigma_x:
- Quantization axis angle: theta = arctan(Delta/epsilon)
- Non-adiabatic transition probability ~ (d_theta/dt)^2 / Omega^2

The Ramen pulse is derived (in Mathematica, per code comments) so that **d_theta/dt follows an arctan(acceleration*t) profile**. This means:
1. The angular velocity is bounded and saturates smoothly
2. No discontinuities in any derivative (unlike trapezoid corners)
3. The sweep rate automatically accounts for the nonlinear relationship between flux amplitude epsilon(t) and quantization angle theta

### square_ramen_pulse — The CZ-Relevant Variant

`square_ramen_pulse(width, ramen_width, target, ...)` splits the Ramen shape at its midpoint:
- First half of Ramen -> rise ramp
- Flat top at `target` for (width - ramen_width) samples
- Second half of Ramen -> fall ramp

This is structurally identical to a trapezoid (rise + flat + fall) but with Ramen-shaped ramps instead of linear ramps. The `ramen_width` parameter controls the total rise+fall duration (analogous to 2*sigma in trapezoid).

---

## Relevance to DTC CZ Gate

### Why Ramen Could Reduce Leakage

Our dominant CZ gate error is **leakage (~3.9%)** from non-adiabatic transitions at the |11⟩ <-> |20⟩ avoided crossing. Current trapezoid pulse has:

1. **Linear ramps** with discontinuous d(epsilon)/dt at corners -> broadband spectral content -> can excite nearby transitions
2. **No crossing awareness** — rise time is a single number (10 ns), not adapted to the level structure
3. **Crude compromise** — 10 ns balances "fast through parasitic crossings" vs "don't kick too hard"

The Ramen pulse addresses all three:
1. **Smooth angular velocity** — arctan profile has no discontinuities at any order
2. **Crossing-aware** — `delta` parameter encodes the |11⟩-|20⟩ gap; pulse shape adapts near the crossing
3. **Tunable aggressiveness** — `acceleration` parameter provides a continuous knob

### Mapping DTC Parameters to Ramen Parameters

To use `square_ramen_pulse` for our CZ gate:
- `target` -> phi_interaction (the flux excursion depth, currently ~0.22 for 296 ns gate)
- `det0` -> detuning of |11⟩ from |20⟩ at idle flux (extract from energy spectrum)
- `delta` -> |11⟩-|20⟩ avoided crossing gap (extract from energy spectrum)
- `acceleration` -> sweep tuning knob (optimize via simulation)
- `ramen_width` -> total rise+fall time (replaces 2 * rise_time = 20 ns)

### Caveats

1. **Single-crossing optimization**: Ramen is designed for ONE avoided crossing. Our flux trajectory passes through multiple crossings. If |11⟩-|20⟩ dominates leakage (it does), this should still help.
2. **Parameter extraction needed**: Must extract det0 and delta from the dressed energy spectrum at the relevant flux points.
3. **Not a silver bullet for decoherence**: Ramen reduces coherent leakage error. Decoherence errors (coupler T1/T_phi away from sweet spot) are unaffected and become the next bottleneck.

### Related: Modulated Variants

- `top_ramen_modu` adds sinusoidal modulation (amplitude `amp1`, frequency `f`) to the Ramen envelope for additional leakage suppression — similar in spirit to DRAG modulation.
- `drag_modu` does the same for Gaussian DRAG pulses.
- These could be explored as a second-order improvement after baseline Ramen.
