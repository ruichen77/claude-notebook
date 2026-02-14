# Pulse Shapes for CZ Gate Simulation — Implementation Spec

**Purpose:** Add these six pulse shapes to `pulse.py` for flux-actuated CZ gate benchmarking in the DTC architecture. Each section gives the mathematical definition, key parameters, and implementation notes.

All pulses define a normalized flux trajectory `Φ(t)` on `t ∈ [0, T_gate]`. The code agent should follow the same interface as existing shapes (square, slepian, cosine, trapezoid): a function returning an array of flux amplitudes given `T_gate`, `dt`, and shape-specific kwargs.

---

## 1. Net-Zero (NZ) Bipolar Pulse

**Reference:** Rol et al., PRL 123, 120502 (2019); Negîrneac et al. (SNZ variant)

**Why it matters:** Destructive interference between two lobes suppresses leakage to ~0.1%. Built-in echo rejects low-frequency flux noise. Zero net flux area makes it robust to long-timescale line distortion. The Li et al. DTC paper (PRX 14, 041050) and the CSDTC paper (Li et al., PRA 2025) both use a "biased net-zero" variant on exactly our architecture.

### Definition

Concatenate two half-pulses of opposite polarity, each shaped by a Slepian (or any base envelope):

```
Φ(t) = A · S(t)              for 0 ≤ t < T_gate/2
Φ(t) = -A · S(T_gate - t)    for T_gate/2 ≤ t ≤ T_gate
```

where `S(t)` is a Slepian-based (DPSS, order 0) envelope on `[0, T_gate/2]`, normalized so `max(S) = 1`.

The net flux area constraint is:

```
∫₀^{T_gate} Φ(t) dt = 0
```

### Variant: Sudden Net-Zero (SNZ)

Insert a short idle gap `t_idle` at the center (between the two lobes) where `Φ = 0`. This allows partial population transfer to interfere more cleanly.

```
Φ(t) = A · S_half(t)                                for 0 ≤ t < T_half
Φ(t) = 0                                            for T_half ≤ t < T_half + t_idle
Φ(t) = -A · S_half(T_gate - t_idle - t)             for T_half + t_idle ≤ t ≤ T_gate
```

where `T_half = (T_gate - t_idle) / 2`. Typical `t_idle` ~ 4–8 ns.

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `A` | Peak flux amplitude (optimize per gate time) | — |
| `t_idle` | Center gap for SNZ variant | 0 ns (NZ) or 4–8 ns (SNZ) |
| `base_shape` | Envelope for each half-pulse | `'slepian'` or `'cosine'` |

---

## 2. Chebyshev-Based Trajectory

**Reference:** Ding et al., Phys. Rev. Appl. 23, 064013 (2025) — MIT/Lincoln Lab

**Why it matters:** Analytically designed to minimize spectral leakage near the anticrossing frequency. ~6% lower leakage than Slepian at equivalent gate times. Formulated as a signal-processing trajectory design problem.

### Definition

The flux trajectory is parametrized using Chebyshev polynomials of the first kind:

```
Φ(t) = A · Σ_{k=0}^{N} c_k · T_k(2t/T_gate - 1)
```

where `T_k(x)` is the k-th Chebyshev polynomial on `[-1, 1]`.

**Practical implementation** — use a Dolph-Chebyshev window:

```python
from scipy.signal import windows
envelope = windows.chebwin(N_samples, at=60)  # at = sidelobe attenuation in dB
envelope = envelope / envelope.max()           # normalize
pulse = A * envelope
```

The `at` parameter (sidelobe attenuation) controls the leakage-vs-gate-time tradeoff:
- Higher `at` → lower leakage but wider main lobe (slower gate)
- Lower `at` → faster gate but more spectral leakage

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `A` | Peak flux amplitude | — |
| `at` | Sidelobe attenuation (dB) for Dolph-Chebyshev | 40–80 dB |
| `coeffs` | Chebyshev polynomial coefficients (alternative) | `[1, 0, -0.5, ...]` |

---

## 3. PiCoS (Piecewise-Constant-Slope) Pulse

**Reference:** arXiv 2412.17454 (Dec 2024) — IBM Zurich / ETH

**Why it matters:** Low-dimensional parametrization (~7–12 params) but rich enough to suppress leakage below 0.3% gate error. Asymmetric shoulders emerge naturally from optimization, intrinsically correcting flux-line distortions.

### Definition

The pulse is defined by a sequence of linear segments with specified breakpoints and slopes:

```
Φ(t) = Φ_i + m_i · (t - t_i)    for t_i ≤ t < t_{i+1}
```

where:
- `{t_0, t_1, ..., t_N}` are breakpoint times with `t_0 = 0`, `t_N = T_gate`
- `Φ_i = Φ(t_i)` is the flux value at breakpoint `i`
- `m_i = (Φ_{i+1} - Φ_i) / (t_{i+1} - t_i)` is the slope of segment `i`

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `breakpoints` | List of `(time_frac, amp_frac)` tuples | 5–9 breakpoints |
| `A` | Overall amplitude scale | — |

---

## 4. Fourier-Series Pulse

**Reference:** arXiv 2412.17454 + Phys. Rev. Applied (2025) — same group

**Why it matters:** 99.9% CZ fidelity in 64 ns with only 7 parameters. Fourier basis naturally produces smooth, bandwidth-limited pulses. Easy to optimize with CMA-ES or GRAPE.

### Definition

```
Φ(t) = A · Σ_{k=1}^{N} a_k · sin(2πk·t/T_gate)
```

This automatically satisfies `Φ(0) = Φ(T_gate) = 0`.

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `A` | Peak amplitude | — |
| `coeffs` | Sine coefficients `[a_1, ..., a_N]` | N = 5–9 |

---

## 5. Flat-Top Raised Cosine (Cosine-Ramped Trapezoid)

**Reference:** arXiv 2511.01260 — TIP gate paper (2025)

**Why it matters:** Simple baseline that achieved 99.7% CZ fidelity. Essentially the trapezoid with cosine rise/fall.

### Definition

```
Φ(t) = A · ½[1 - cos(πt/t_rise)]                  for 0 ≤ t < t_rise
Φ(t) = A                                            for t_rise ≤ t < T_gate - t_fall
Φ(t) = A · ½[1 + cos(π(t - T_gate + t_fall)/t_fall)]  for T_gate - t_fall ≤ t ≤ T_gate
```

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `A` | Flat-top amplitude | — |
| `t_rise` | Cosine ramp-up duration | 10–30 ns |
| `t_fall` | Cosine ramp-down duration (default = `t_rise`) | 10–30 ns |

---

## 6. RL/GRAPE-Optimized Free-Form Pulse

**Reference:** Li et al., PRX 14, 041050 (2024) — DTC paper

**Why it matters:** Upper-bound reference. Model-free optimization achieved 99.90% CZ fidelity at 48 ns on the DTC architecture — our exact system.

### Definition

The pulse is discretized into `M` time bins of width `Δt`:

```
Φ(t) = Φ_m    for m·Δt ≤ t < (m+1)·Δt,   m = 0, 1, ..., M-1
```

where `{Φ_0, Φ_1, ..., Φ_{M-1}}` are free parameters optimized to minimize a cost function.

### Parameters
| Param | Description | Typical |
|-------|-------------|---------|
| `M` | Number of time bins | `T_gate / dt` |
| `bandwidth_GHz` | Smoothing bandwidth | 0.3–0.5 GHz |
| `lambda_leak` | Leakage penalty weight | 1–10 |

---

## Recommended Simulation Priority

| Priority | Shape | Params | Rationale |
|----------|-------|--------|-----------|
| **1** | Net-Zero bipolar | 2–3 | Directly used on DTC hardware (Li et al.). Tests leakage interference physics. |
| **2** | Chebyshev window | 2 | Analytically superior to Slepian. Drop-in comparison. |
| **3** | Flat-top raised cosine | 2 | Clean baseline — your trapezoid with smooth corners. |
| **4** | Fourier series | 5–9 | Bridge to optimization. Start from flat-top-like init. |
| **5** | PiCoS | 7–12 | If Fourier works, PiCoS adds distortion correction capability. |
| **6** | Free-form (GRAPE) | ~50 | Upper bound. Run last — most compute, most insight. |

---

## Key References

1. **Rol et al.** — "Fast, High-Fidelity Conditional-Phase Gate Exploiting Leakage Interference" — PRL 123, 120502 (2019)
2. **Negîrneac et al.** — "High-Fidelity CZ Gate with Maximal Intermediate Leakage Operating at the Speed Limit" — Phys. Rev. Lett. 126, 220502 (2021)
3. **Ding et al.** — "Pulse Design of Baseband Flux Control for Adiabatic Controlled-Phase Gates" — Phys. Rev. Appl. 23, 064013 (2025)
4. **IBM Zurich / ETH** — "Sensitivity-Adapted Closed-Loop Optimization for High-Fidelity CZ Gates" — arXiv 2412.17454 (2024)
5. **Li et al.** — "Realization of High-Fidelity CZ Gate Based on a Double-Transmon Coupler" — Phys. Rev. X 14, 041050 (2024)
6. **Li et al.** — "Capacitively Shunted DTC Realizing Bias-Free Idling and High-Fidelity CZ Gate" — Phys. Rev. Appl. (2025)
7. **TIP gate** — "High-fidelity all-microwave CZ gate with partial erasure-error detection via a transmon coupler" — arXiv 2511.01260 (2025)
