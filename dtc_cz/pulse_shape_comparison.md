# Pulse Shape Comparison: qss-pulsegen vs dtc_cz_sim

Comparison of three pulse shapes between the hardware library (qss-pulsegen) and our simulation code (dtc_cz_sim). Created Feb 18, 2026.

## Summary of Differences

| Pulse | qss-pulsegen | dtc_cz_sim | Match? |
|-------|-------------|------------|--------|
| **trapezoid** | Linear ramp (`np.linspace`) | **Cosine ramp** (`0.5*(1-cos)`) | **NO — different shape** |
| **ramen_pulse** | Same math | Same math (ported) | YES |
| **square_ramen_pulse** | Same concept | Same concept (ported) | YES (minor details differ) |

### CRITICAL: "trapezoid" is misnamed in dtc_cz_sim

Our `trapezoid_pulse` uses cosine-shaped edges (raised cosine / Hann window), NOT linear ramps. This means:
- All simulation results labeled "trapezoid" are actually **flat-top cosine** pulses
- A true trapezoid (linear ramps) has derivative discontinuities at rise↔flat transitions
- The real hardware trapezoid (qss-pulsegen) uses `np.linspace` = linear ramp

---

## 1. Trapezoid

### qss-pulsegen (`trapezoid`)
```python
# Rise: LINEAR ramp
on = np.linspace(0, amp, sigma)      # sigma = rise duration in samples
off = np.linspace(amp, 0, sigma)     # LINEAR fall
# Full pulse: [linear_rise | flat_top | linear_fall]
return mult * np.concatenate((on, np.ones(width - 2*sigma) * amp, off))
```

**Interface**: Sample-based. `width` = total samples, `sigma` = rise/fall samples, `amp` = amplitude.

**Key properties**:
- φ(t) is piecewise linear (true trapezoid shape)
- dφ/dt is piecewise constant, with **discontinuities** at rise↔flat and flat↔fall transitions
- d²φ/dt² has delta-function spikes at the kinks
- This is what the hardware AWG actually outputs

### dtc_cz_sim (`trapezoid_pulse`)
```python
# Rise: COSINE ramp
envelope[m_rise] = 0.5 * (1 - np.cos(np.pi * t / rise))   # raised cosine
# Fall: COSINE ramp
envelope[m_fall] = 0.5 * (1 + np.cos(np.pi * (t - t_fall_start) / rise))
```

**Interface**: Continuous function of time. `gate_time` (ns), `phi_idle`, `phi_interaction`, `rise_time` (ns).

**Key properties**:
- φ(t) is C∞ smooth everywhere (cosine has all derivatives continuous)
- dφ/dt follows a half-sine profile: zero at edges, peak at midpoint of rise
- Peak dφ/dt = (π/2) × average dφ/dt
- **NOT a trapezoid** — this is a flat-top cosine (Hann-windowed edges)

### Implications
- Our "trapezoid" results (3.7% leakage at T=100ns) are for a cosine-ramped pulse, not the hardware trapezoid
- A true linear trapezoid might perform differently due to sharper crossing traversal
- The adaptive ODE solver may need more evaluations near linear kinks (step size reduction)

---

## 2. Ramen Pulse (Full Arch)

### qss-pulsegen (`ramen_pulse`)
```python
t = np.linspace(-np.pi, np.pi, width)  # width = total samples
# Ramen shape: detuning trajectory that makes quantization axis
# rotate at rate ~ arctan(acceleration * t)
ps = det0 + delta * np.tan(
    (A * g(t) + B * h(t)) / D
)
# where:
#   A = arctan((det0 - target) / delta)
#   B = arctan(det0 / delta)
#   g(t) = -2a*pi*arctan(a*pi) + 2a*t*arctan(a*t) + log(1+a²π²) - log(1+a²t²)
#   h(t) = -2a*t*arctan(a*t) + log(1+a²t²)
#   D = 2a*pi*arctan(a*pi) - log(1+a²π²)
return ps * mult  # returns DETUNING values (target units)
```

**Interface**: Sample-based. `width` = samples, `target` = peak detuning, `det0` = idle detuning, `acceleration`, `delta` = crossing gap. Returns detuning values directly.

### dtc_cz_sim (`ramen_pulse`)
```python
tau = -np.pi + 2 * np.pi * t_m / gate_time   # map t -> [-pi, pi]
target = phi_interaction - phi_idle            # amplitude in flux units

# SAME math as qss-pulsegen:
ramen_shape = det0 + delta * np.tan((A * g_tau + B * h_tau) / D)

phi[mask] = phi_idle + ramen_shape  # add idle offset -> return flux values
```

**Interface**: Continuous function of time. `gate_time` (ns), `phi_idle`, `phi_interaction`, `delta` (GHz), `det0` (GHz), `acceleration`. Returns flux values.

### Comparison
- **Math is identical** — same formula, ported faithfully
- **Coordinate mapping differs**:
  - qss-pulsegen: `t = np.linspace(-pi, pi, width)` (sample indices → [-π, π])
  - dtc_cz_sim: `tau = -pi + 2*pi*t/gate_time` (physical time in ns → [-π, π])
- **Output units differ**:
  - qss-pulsegen: returns detuning values (`target` units, typically MHz or GHz)
  - dtc_cz_sim: returns flux values (`phi_idle + ramen_shape`)
- **Parameter mapping**:
  - qss-pulsegen `target` = dtc_cz_sim `phi_interaction - phi_idle`
  - qss-pulsegen `det0`, `delta`, `acceleration` = same in both

---

## 3. Square Ramen Pulse

### qss-pulsegen (`square_ramen_pulse`)
```python
# Generate full Ramen arch over ramen_width samples
ramen_ps = ramen_pulse(ramen_width, target, det0, acceleration, delta)
# Split at midpoint
ramen_halfway_idx = int(ramen_width / 2)
square_height = ramen_ps[ramen_halfway_idx]  # or average of neighbors if even
# Assemble: [ramen_rise | flat_top_at_square_height | ramen_fall]
return np.concatenate((
    ramen_ps[:ramen_halfway_idx],
    square_height * np.ones(width - ramen_width),
    ramen_ps[ramen_halfway_idx:],
))
```

**Interface**: Sample-based. `width` = total samples, `ramen_width` = rise+fall samples.
Also has chunked version for hardware upload (splits into sub-pulses for AWG memory).

### dtc_cz_sim (`square_ramen_pulse`)
```python
# Generate full Ramen arch over ramen_width (ns)
n_ramen = max(1001, int(ramen_width * 10))
t_ramen = np.linspace(0, ramen_width, n_ramen)
phi_ramen = ramen_pulse(t_ramen, ramen_width, phi_idle, phi_interaction, ...)

# Split at midpoint
mid_idx = n_ramen // 2
flat_val = phi_ramen[mid_idx]

# Interpolate for continuous evaluation
envelope[m_rise] = np.interp(t_m[m_rise], t_rise, rise_ramen)
envelope[m_flat] = flat_val
envelope[m_fall] = np.interp(t_local, t_fall, fall_ramen)
```

**Interface**: Continuous function of time. `gate_time` (ns), `ramen_width` (ns) = total rise+fall duration.

### Comparison
- **Same concept**: Split full Ramen arch at midpoint, insert flat top
- **dtc_cz_sim uses interpolation** (`np.interp`) to evaluate at arbitrary times — this adds slight smoothing at the rise→flat and flat→fall transitions (the Ramen shape at its midpoint has zero derivative anyway, so this is benign)
- **Flat top value**: Both use the Ramen midpoint value as the flat-top height
- **`ramen_width` meaning**: Same in both — total rise+fall duration (rise = ramen_width/2)

---

## Parameter Mapping Reference

| qss-pulsegen | dtc_cz_sim | Notes |
|-------------|------------|-------|
| `width` (samples) | `gate_time` (ns) | `width = gate_time * sample_rate` |
| `amp` or `target` | `phi_interaction - phi_idle` | qss returns detuning; dtc returns flux |
| `sigma` (samples) | `rise_time` (ns) | Trapezoid rise duration |
| `ramen_width` (samples) | `ramen_width` (ns) | Square Ramen rise+fall duration |
| `det0` | `det0` | Idle detuning |11⟩-|20⟩ (GHz in dtc) |
| `delta` | `delta` | Avoided crossing gap (GHz in dtc) |
| `acceleration` | `acceleration` | Arctan profile steepness |

---

## Action Items

1. **Rename or fix `trapezoid_pulse`** in dtc_cz_sim:
   - Option A: Rename to `cosine_flat_top_pulse` (accurate name)
   - Option B: Add a true `linear_trapezoid_pulse` with `np.linspace` ramps
   - Option C: Both — keep cosine version but add the linear one for comparison
2. **Run comparison**: True linear trapezoid vs cosine "trapezoid" at T=100ns to quantify the difference
3. **Update all docs/presentations**: Clarify that "trapezoid" results used cosine ramps
