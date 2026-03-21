# Cooldown: 20260223_RANNCTHRU_RANV505HS

**Server**: timtam
**Fridge**: fridge36B
**Path**: `/nas-data1/systems/fridge36B/20260223_RANNCTHRU_RANV505HS/`
**Date**: 2026-02-23

## Wiring: Device Table

| Throw | Device | Type | Switch | Pump | Notes |
|-------|--------|------|--------|------|-------|
| throw1 | HDRP094A | TWPA (G + 3dB pad) | AB1 | PUMP01 | Shared pump line |
| throw2 | RANNCTRHU-D1 | TWPA (THRU-NC-BPF) | AB2 | PUMP01 | Shared pump line |
| throw3 | RANV505-D6 | TWPA | AB3 | PUMP06 | F1 / HWch6 |
| throw4 | RANNCTHRU-D8 | TWPA (DPX-NC-THRU) | AB4 | PUMP04 | F21 / Pump2 |
| throw5 | HDR137-D1 | TWPA (G + 3dB pad) | AB5 | PUMP01 | Shared pump line |
| throw6 | nestedthru | Thru reference | AB6 | — | Short thru (Rainier mounted) |

## Pump Sources

| Pump | Instrument | Serves | Physical Port |
|------|-----------|--------|---------------|
| PUMP01 | Psg242 (E8257D) | throw1 (HDRP094A), throw2 (RANNCTRHU-D1), throw5 (HDR137-D1) | Moved from P3 → F1 |
| PUMP06 | Holzworth HS9000 ch6 | throw3 (RANV505-D6) | F1 / HWch6 |
| PUMP04 | psg-80784 (E8257D) | throw4 (RANNCTHRU-D8) | F21 |

## Signal Chain

Input: BF noise source → SW B (input attenuation ~2.3 dB)
Output: SW A → LPF → SC coax 4 (~150 cm) → HEMT2 → Out 15

## Wiring Changes Applied

- [x] Change pump line for throw4: bluegreen 2 → bluegreen 1
- [x] Move pump1 from P3 to F1
- [ ] Mount ardent

## Measurement Progress

### Unpumped S21 (2026-02-23)
- Collected using `run_unpumped_s21.sh` automation script
- Reference: `nestedthru_ref` at AB6
- All 5 DUTs collected (background-subtracted)
- Overlay plot: `benchmark/vna_scan/unpumped_s21_overlay.html`
