# Cooldown Plan: 20260223_RANNCTHRU_RANV505HS

**Fridge**: fridge36B (timtam)
**Path**: `/nas-data1/systems/fridge36B/20260223_RANNCTHRU_RANV505HS/benchmark/`

## Device Status

| Throw | Device | Status |
|-------|--------|--------|
| throw1 | HDRP094A | Noise temp — check if finished (was ~47% at 08:00 Feb 24) |
| throw2 | RANNCTRHU-D1 | Custom list running (ETA Wed 18:00); extended queued after (~7 hrs) |
| throw3 | RANV505-D6 | **Dead** (manual inspection) — skip |
| throw4 | RANNCTHRU-D8 | **Dead** (manual inspection) — skip |
| throw5 | HDR137-D1 | Irrelevant for current goals — skip |
| throw6 | nestedthru | Thru reference — done |

## Measurement Plan

### 1. HDRP094A noise temp (in progress)
- Config: `config_HDRP094A.yaml`
- Pump: 8.13 GHz, 4.0-7.4 dBm (0.2 dBm step, 18 powers)
- Temps: 0.2, 0.3, 0.4, 0.5 K
- Probe: 6.5-7.5 GHz (21 freqs)
- Status: ~47% done as of Feb 24 08:00, ETA ~13:30

### 2. RANNCTRHU-D1 extended sweep (next)
- Goal: extend to lower gain region for complete QE vs Gain curve
- Pump: 8.06 GHz, **12.0-15.0 dBm, 0.2 dBm step** (16 powers)
- Temps: 0.2, 0.3, 0.4, 0.5 K
- Probe: 6.5-7.5 GHz (21 freqs)
- Total points: 4 × 16 × 21 = 1344
- Estimated time: ~8-10 hrs

### 3. SA spectrum collection
- Raw SA traces at best QE pump conditions per device
- Quick diagnostic before warmup

## Timeline

| When | Task |
|------|------|
| Tue afternoon | HDRP094A finishes (check) |
| Tue PM → Wed PM | RANNCTRHU-D1 custom list sweep (50 configs, 4200 pts) |
| Wed PM → Thu AM | RANNCTRHU-D1 extended sweep resume (12-15 dBm, 1229 pts remaining) |
| Thu | Analysis of both sweeps |
| Fri EOD | **Warm up** |

## Next Cooldown

- Target: **Mon Mar 2** if next RANNCTHRU sample ready (check with Shayne)
- Vacation starts **Wed Mar 5**

## Analysis Completed

- RANNCTRHU-D1 individual plots: `benchmark/analysis_RANNCTRHU_D1/`
- Multi-device comparison (RANNCTRHU-D1, HDRP094A, RANV402-D1/D7, RANV502-D1/D7):
  - Local: `~/projects/twpa/sim_outputs/20260224_noise_analysis/`
  - Scripts: `multi_device_comparison.py`, `cross_device_comparison.py`
