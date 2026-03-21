# Cooldown: 20260318_RANV5BPF_RANCAL401

**Server**: timtam
**Fridge**: fridge36B
**Path**: `/nas-data1/systems/fridge36B/20260318_RANV5BPF_RANCAL401/`
**Date**: 2026-03-18

## Wiring: Device Table

| Throw | Device | Type | Switch | Pump | Notes |
|-------|--------|------|--------|------|-------|
| throw1 | HDRP094A | TWPA (G + 3dB pad) | AB1 | PUMP01 | Carried over |
| throw2 | RANV5BPF-D1 | TWPA (THRU-NC-BPF, v5LLD) | AB2 | PUMP01 | Group A |
| throw3 | RANCAL401-D8 | TWPA (Cal2P045-Thru, high freq NC) | AB3 | PUMP01 | F3 not connected at room temp |
| throw4 | RANV5BPF-D8 | TWPA (DPX-NC-THRU, V5LLD) | AB4 | PUMP01 | Group A |
| throw5 | RANCAL401-D1 | TWPA (Cal2P045-BPF) | AB5 | PUMP01 | Group B |
| throw6 | nestedthru | Thru reference (short thru) | AB6 | — | Rainier sc integrated twpa mounted |

## Device Groups

- **Group A (RANV5BPF)**: D1 = BPF-v5LLD-BPF, D8 = BPF-V5LLD-Thru
- **Group B (RANCAL401)**: D1 = BPF-Cal2P045-BPF, D8 = BPF-Cal2P045-Thru (high freq NC)

## Pump Sources

| Pump | Instrument | Serves | Physical Port |
|------|-----------|--------|---------------|
| PUMP01 | Psg242 (E8257D) | ALL devices (throw1–5) | F1 (shared pump line) |

All devices use the shared PUMP01 line this cooldown.

## Signal Chain

Input: BF noise source → SW B (input attenuation ~2.3 dB)
Output: SW A → LPF → SC coax 4 (~150 cm) → HEMT2 → Out 15

## Wiring Changes Applied (from previous cooldown)

- [x] Change pump line for throw4: bluegreen 2 → bluegreen 1
- [x] Move pump1 from P3 to F1
- [ ] Mount ardent

## exp_params.yaml Device Names

| Throw | YAML key | Switch relay |
|-------|----------|-------------|
| throw1 | HDRP094A | [1] |
| throw2 | RANV5BPFD1 | [2] |
| throw3 | RANCAL401D8 | [3] |
| throw4 | RANV5BPFD8 | [4] |
| throw5 | RANCAL401D1 | [5] |
| throw6 | nestedthru | [6] |

## Measurement Progress

_(none yet)_
