# Ramp Speed Study: Square Ramen Leakage vs Ramp Time

**Date**: Feb 27, 2026
**Notion**: https://www.notion.so/3146a7182abd8179aeb0f7426f18c78b

## Results (75 ns gate, adiabatic_cz with fixed propagator)

| Pulse | Ramp Time | Flat Top | Fidelity | Avg Leakage | Amplitude (Φ₀) |
|:--|:-:|:-:|:-:|:-:|:-:|
| Full Ramen | — | — | 0.999961 | 0.0004% | 0.1828 |
| Square Ramen | 30 ns | 15 ns | 0.9992 | 0.005% | 0.1414 |
| Square Ramen | 20 ns | 35 ns | 0.9989 | 0.009% | 0.1362 |
| Square Ramen | 10 ns | 55 ns | 0.9978 | 0.07% | 0.1362 |
| Square Ramen | 2 ns | 71 ns | 0.9887 | 0.57% | 0.1259 |

## Key Findings
- Leakage increases ~1400x from full Ramen to near-square (rw=2ns)
- rw=20ns is a good compromise: 0.009% leakage with 35ns flat top
- Full Ramen is leakage-optimal (smooth arch, no sharp transitions)
- Conditional phase ~0.5π at best amplitude (30-point grid too coarse for exact π)

## Script & CLI
`tuned_cz_demo.py` now has argparse:
- `--cores 21` (default 21)
- `--pulse-type ramen|square_ramen` (default ramen)
- `--ramen-width <ns>` (rise/fall time, converted to µs internally)
- `--gate-time <ns>` (default 75)
- `--amp-range <lo> <hi>` (default 0.10 0.25)

Shell runner: `examples/run_ramp_study.sh` — loops over ramen_widths sequentially, reuses cached eigenbasis.

## Data Location
- Remote: `landsman2:.../dtc_adiabatic_sim/results/20260227_*`
- Local: `~/projects/dtc/sim_outputs/dtc_adiabatic_sim/20260227_*`
- Old (buggy, pre-fix): moved to `results_old_pre_fix/` on landsman2

## Bug Fix: Lost Commits Recovered
5 commits (M matrix fix + RF→lab-frame fix + viz improvements) were lost from landsman2
due to `git reset --hard origin/master`. Recovered from reflog via cherry-pick and pushed.
New hashes: `3f9094c`, `9d21b24`, `399a560`, `ed6aa5f`, `0e631f9`.
The initial run with the buggy code showed 2.8-5.3% leakage (vs correct 0.0004-0.57%).

## TODO / Next Steps
- [ ] Finer amplitude tuning (brentq) to hit exact π phase
- [ ] Add decoherence (Lindblad with coupler T1)
- [ ] Compare with hardware CZ gate data
- [ ] Sweep gate time (e.g., 50, 100, 150 ns) at fixed ramp time
- [ ] Cross-validate rw=2ns (square) with duffing_cz square pulse result
