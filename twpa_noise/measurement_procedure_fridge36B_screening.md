# Measurement Procedure — fridge36B Screening Setup (timtam)

**Last verified**: 2026-04-12, Cooldown 20260408_RANCAL203_RANCAL202

---

## Prerequisites

- SSH access to timtam with ControlMaster
- HEMT biased (DO NOT TOUCH — check with operator which HEMT serves screening vs TRL)
- Cooldown folder exists at `/nas-data1/systems/fridge36B/<cooldown_name>/`
- `switch_map.yaml` in cooldown folder with device-to-switch mappings

## Directory Structure

```
<cooldown>/
├── switch_map.yaml                    # device → switch mapping (created at cooldown setup)
├── benchmark/
│   ├── instruments.yaml               # instrument addresses (copy from prev cooldown)
│   ├── ibmqc.yaml                     # experiment framework config (copy from prev cooldown)
│   ├── vna_scan/
│   │   ├── record_trace.yaml          # VNA config: state_file, instruments, variables
│   │   ├── exp_params.yaml            # device definitions with switch_map entries
│   │   ├── collect_unpumped_s21.sh    # batch script
│   │   └── 2026-mm-dd_HH.MM.SS_*/    # output directories (one per trace)
│   └── gain_map/
│       ├── record_trace.yaml          # pumped mode config
│       ├── exp_params.yaml            # same device definitions
│       ├── collect_gain_maps.sh       # coarse scan batch
│       ├── collect_fine_scan.sh       # fine scan batch
│       └── fine_plots/                # generated heatmaps + noise plan
```

## Tool: record_trace.py

**Location**: `/nas-data0/systems/fridge36B/recordtrace/record_trace.py`

**record_trace.yaml** is the single master config — contains:
- `experiment` block: sweep_instrument, sweep_attribute, sweep_range, state_file, trace_to_record
- `instruments` block: all RF sources, VNA, SA with addresses and variable bindings
- `variables` block: default values for all pump freqs, powers, phases, outputs

**Key state files** (on VNA at `D:\Ruichen_cals\`):
- `screening_thru_cal_5-8p5_1000_pts.csa` — standard screening VNA state (5–8.5 GHz, 1000 pts)
- `screening_thru_cal_S43.csa` — alternate port config
- `SA_D_5-9GHz.csa` — spectrum analyzer mode

**Invocation**:
```bash
cd <working_directory>   # must contain record_trace.yaml + exp_params.yaml
python /nas-data0/systems/fridge36B/recordtrace/record_trace.py -dn <output_dirname>
```

**Override variables before each run**:
```bash
params set <variable_name> <value>
# Examples:
params set sweep_instrument PUMP04
params set sweep_attribute power
params set sweep_range 'np.arange(-10,0.25,0.25)'
params set PUMP04_freq 8.4e9
params set PUMP04_out 1
params set PUMP01_out 0
params set background_data '/path/to/thru_reference.csv'
```

## Switch Control

**Signal switches**: `set_switches <name>` using names from `switch_map.yaml`
```bash
set_switches AB2          # direct AB chain throw
set_switches RANCAL203-D6 # compound: sets AB + CD switches
```

**CD chain workaround**: When routing to CD devices, set CD switch FIRST, then AB1 separately. Simultaneous 4-switch latching causes intermittent relay faults.
```bash
set_switches CD1    # set SMM-C/D first
set_switches AB1    # then route AB into CD chain
```

**Pump multiplexer** (RT SP8T, Mini-Circuits RCM-2SP8T):
```bash
set_switches PUMP04_to_P2   # routes PUMP04 → cryo pump line P2
set_switches PUMP04_to_F21  # routes PUMP04 → cryo pump line F21
```

**Switch retry protocol**: If `set_switches` fails with SetError, wait 5 minutes and retry once. If still fails, it's a hardware fault — skip that device and note in measurement log.

## Pump Safety

**CRITICAL: Only one pump source ON at a time.**

- RANCAL devices use PUMP04 (via SP8T mux)
- HDRP094A devices use PUMP01 (shared direct line)
- Before switching pump source, ALWAYS turn off the old one first:
```bash
params set PUMP04_out 0    # turn off PUMP04
params set PUMP01_out 1    # THEN turn on PUMP01
```
Both on simultaneously = overdriven TWPA, gain crashes.

## Phase 1: Unpumped S21

**Purpose**: Measure passive insertion loss / bandpass shape of each device.

**record_trace.yaml config** (unpumped mode):
```yaml
experiment:
    sweep_instrument: PUMP01
    sweep_attribute: freq
    sweep_range: '[8e9]'          # single point, pump off
    state_file: "D:\\Ruichen_cals\\screening_thru_cal_5-8p5_1000_pts.csa"
    trace_to_record: [1]
```

**Procedure**:
1. `all_rf_off` — turn off all RF sources
2. Collect thru references FIRST (no background subtraction):
   - `short_thru_AB` (AB6) for AB chain
   - `short_thru_CD` (AB1→CD6) for CD chain
3. Set `background_data` to the thru CSV path via `params set`
4. Collect each DUT — AB devices use AB thru, CD devices use CD thru
5. Each trace produces one CSV: `sweep_PUMP01_freq_setValTo_8e9_param_S21_magnitude_phase.csv`

**Output columns**: `f`, `S21_magnitude`, `S21_phase`

**Gain = raw S21** (record_trace subtracts background_data automatically when set).

## Phase 2: 2D Gain Map (Coarse)

**Purpose**: Find gain onset, optimal pump frequency and power per device.

**record_trace.yaml config** (pumped mode):
```yaml
experiment:
    sweep_instrument: $sweep_instrument
    sweep_attribute: $sweep_attribute
    sweep_range: $sweep_range
    state_file: "D:\\Ruichen_cals\\screening_thru_cal_5-8p5_1000_pts.csa"
    trace_to_record: [1]
```

**Typical coarse parameters**:
- RANCAL: fpump 8.0–8.5 GHz (50 MHz, 11 pts) × pp -10 to 0 dBm (1 dB, 11 pts)
- HDRP094A: fpump 7.8–8.25 GHz (50 MHz, 10 pts) × pp 4–14 dBm (1 dB, 11 pts)
- ~4–5 min per device at 2.5 sec/point

**Batch script pattern** (outer loop = fpump in bash, inner sweep = pp in record_trace):
```bash
params set sweep_instrument PUMP04
params set sweep_attribute power
params set sweep_range 'np.arange(-10,0.25,1)'
params set PUMP04_out 1
params set PUMP01_out 0
params set background_data "$THRU_CSV"

set_switches AB2
set_switches PUMP04_to_P2

for freq in 8.0e9 8.05e9 8.1e9 ...; do
    params set PUMP04_freq $freq
    python $recordtrace -dn RANCAL203_D1_fpump_${freq}
done
```

## Phase 3: 2D Gain Map (Fine)

**Purpose**: High-resolution gain landscape for noise measurement planning.

**Typical fine parameters**:
- Freq: 25 MHz steps (27 pts for RANCAL, 19 for HDRP094A)
- Power: 0.25 dB steps
- ~42 min per RANCAL device, ~36 min per HDRP094A

## Phase 4: Noise Temperature Measurement

**Purpose**: Y-factor noise temp → quantum efficiency at multiple operating points.

**Planning from gain map**:
1. Find "good gain region" in fpump space (where max gain > 10 dB)
2. Pick 5 fpump values uniformly across region, including peak
3. At each fpump, pick pp values giving gain from 3 dB to max in 3 dB steps
4. Always include global max point

**Tool**: `twpa_noise_measurement` package
```bash
python -m noise_measurement run -c config.yaml
```
See `~/.claude/docs/twpa_noise/noise_measurement_package.md` for full docs.

## Plotting

**Dark theme required** (per CLAUDE.md). Generate plots on timtam (faster, no SSH per CSV), copy HTML/PNG to laptop.

**Key plots**:
- Unpumped S21 overlay (all devices, thru-calibrated per chain)
- Readout band zoom (6.5–7.5 GHz, 0 to -10 dB)
- Avg gain vs pump power overlay (y-axis 0–20 dB)
- Combined 2D heatmap grid (all devices on one slide, with ★ peak + ● noise dots)

**PNG generation**: If kaleido not on timtam, generate HTMLs on timtam, SCP to laptop, extract Plotly JSON from HTML and render PNG locally.

## PPTX Updates

Update measurement notes PPTX in Box cooldown folder (`measurementnotes_<date>.pptx`). Use `python-pptx` to add slides. Rebuild from the clean copy (backup) when replacing slides to avoid XML corruption from slide reordering.

## Common Issues

| Issue | Fix |
|---|---|
| `instruments.yaml not found` | Symlink or copy from prev cooldown into working dir |
| `Variable PUMP06_freq not defined` | Add all PUMP variables to record_trace.yaml `variables` block |
| `SetError: Reset relay N -> Set` | Switch relay fault. Wait 5 min, retry. If persistent, skip device. |
| CD switch fault on compound `set_switches` | Set CD first, then AB1 separately |
| `PUMP03 connection timeout` | psg-310 offline. Comment out in instruments if not used. |
| Both pumps on simultaneously | TWPA overdriven, gain crashes. Always toggle pump_out 0/1 explicitly. |
| `kaleido ChromeNotFoundError` | No Chrome on timtam. Generate HTML on timtam, PNG locally. |
