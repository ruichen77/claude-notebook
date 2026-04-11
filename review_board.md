# TWPA Review Board — Enforcement Rules

**Every TWPA simulation, analysis, or investigation must be tracked in the Supabase review board.** No exceptions. Untracked work is lost context — it becomes invisible to SessionStart hook injection, invisible to the diary, invisible to future-you, invisible to collaborators.

## Where the review board lives

- Supabase project: `fqdmbvdaxjslgoqgclip`
- Primary table: `simulation_runs`
- Related tables: `investigation_nodes`, `diary_entries`, `parking_lot`, `beliefs`
- SessionStart hook auto-injects the current state of this board at every session start.

## Rules — when to write to the board

### R1 — CREATE a card before launching ANY simulation ≥ 5 min wall time

Before running a campaign, **INSERT** a row into `simulation_runs`. This must happen *before* the launch, not after. Creating the card is part of launching the sim, not a follow-up.

Minimum required fields:

```sql
INSERT INTO simulation_runs (
  run_id,               -- e.g. 'W14-47', 'W14-47c'. Phase IDs per the twpa campaign numbering convention.
  name,                 -- human-readable title
  campaign,             -- integer (14 for Campaign 14)
  phase,                -- phase letter (e.g. '18C')
  status,               -- 'running' at launch
  board_status,         -- 'in_flight' at launch (allowed: archived, in_flight, key_result, needs_review, reviewed, superseded)
  device_type,          -- 'v3', 'v4', 'v5', 'multi', 'hd3c', etc.
  server,               -- 'landsman5', 'timtam', ...
  server_dir,           -- where outputs go (absolute path)
  script,               -- absolute path to the runner script
  purpose,              -- WHAT we're testing, in plain language
  question,             -- WHAT we'll learn — the specific question this sim answers
  launched_at,          -- NOW()
  estimated_duration,   -- e.g. '2h', '30m' — best guess
  week,                 -- ISO week string e.g. '2026-W15'
  tags                  -- ARRAY of descriptors e.g. ARRAY['crucible','v3_v4','investigation_80']
)
VALUES (...);
```

If the sim launch is scripted (e.g. a crucible campaign), the insert should be in the campaign script itself or a tiny wrapper around it — not a manual step that can be forgotten.

### R2 — UPDATE the card when results land

As soon as the sim completes (or fails), update the card:

- `status` → `completed` or `failed`
- `completed_at` → NOW()
- `actual_duration` → observed wall time
- `result_summary` → 2–3 sentence headline of what the numbers show
- `key_finding` → **one-sentence** bottom-line conclusion (this field is the one that shows up in the injected context)
- `result_files` → jsonb with local + remote paths, INCLUDING any plot HTML files (see R3)
- `local_copy` → absolute path where the local sync of the results lives
- `board_status` → `needs_review` (default after sim done — stays here until the user confirms conclusions)

### R3 — UPDATE `result_files` whenever a new plot / artifact is generated

Every plot you generate from a sim result must be registered in the card's `result_files` jsonb. Format:

```sql
UPDATE simulation_runs
SET result_files = jsonb_build_object(
    'summary_json',      'landsman5:/data/rzhao/<...>/summary.json',
    'local_summary_json','/Users/ruichenzhao/projects/twpa/sim_outputs/<...>/summary.json',
    'plot_html',         'file:///Users/ruichenzhao/projects/twpa/sim_outputs/<...>/plot.html',
    'plot_script',       '/Users/ruichenzhao/projects/twpa/sim_outputs/<...>/make_plot.py'
    -- add additional keys for more plots
  )
WHERE run_id = '<ID>';
```

If the card already has `result_files` and you're adding another plot, use `result_files || jsonb_build_object(...)` instead of overwriting.

### R4 — UPDATE `board_status` at each review milestone

- **in_flight** → set at launch (R1)
- **needs_review** → set when sim completes, before user sees the results
- **reviewed** → set when user has seen the results and either accepted or dismissed them
- **key_result** → set when user explicitly marks the result as important (feeds the "Recent Key Results" context injection)
- **superseded** → set when a newer run invalidates this one. Also fill `superseded_by` with the new `run_id`.
- **archived** → set when no longer relevant (low-priority housekeeping)

### R5 — CROSS-REFERENCE open investigations

If the sim belongs to an open investigation (e.g. "Investigation #80: V3/V4 sim-vs-exp gap"), tag the run with the investigation number:

```sql
tags = ARRAY['investigation_80', ...]
```

Optionally also insert/update a row in `investigation_nodes` if this is a new hypothesis branch — but this is secondary.

### R6 — NEVER run a tracked sim without a card

If you catch yourself launching a sim without a corresponding row in `simulation_runs`, that's a rule violation. Stop the launch, create the card, then launch. The reviewing-and-diary-write pipeline depends on this table.

**Exemption:** one-off smoke tests for Rule A (reasoning rule `--smoke`) that run < 5 min wall time don't need a card. Everything larger does.

## Cheat sheet — one-liners

```sql
-- R1: create card
INSERT INTO simulation_runs (run_id, name, campaign, phase, status, board_status, ...) VALUES (...);

-- R2: mark complete with headline
UPDATE simulation_runs
SET status='completed', board_status='needs_review',
    completed_at=NOW(), actual_duration='<Xh>',
    result_summary='...', key_finding='...',
    local_copy='<path>',
    result_files=jsonb_build_object('plot_html','file://<path>', 'summary_json','<path>')
WHERE run_id='<ID>';

-- R3: append a new plot to an existing card
UPDATE simulation_runs
SET result_files = COALESCE(result_files, '{}'::jsonb) || jsonb_build_object('ripple_plot','file://<path>')
WHERE run_id='<ID>';

-- R4: mark as key result after user confirms
UPDATE simulation_runs SET board_status='key_result' WHERE run_id='<ID>';

-- R4: mark as superseded when invalidated
UPDATE simulation_runs SET board_status='superseded', superseded_by='<new_id>' WHERE run_id='<old_id>';
```

## Why these rules exist

- **SessionStart context injection** reads `simulation_runs` + `investigation_nodes` at every session start. Cards that don't exist are invisible to future-you. Losing continuity between sessions is the #1 reason research slows down.
- **Diary auto-generation** (if the harvester runs) consumes `key_finding` and `result_summary`. Fields left blank degrade the diary.
- **Investigation trees** depend on cards being linked to investigations. Uncross-referenced runs show up as orphaned work.
- **Collaboration / reviewing** — the board is the shared truth of what's run and what it showed. A meeting where someone asks "what did you find?" is answered by querying the board.

## Canonical source

This file is also mirrored at `~/context-engine/templates/twpa-review-board-rules.md` for cross-machine deployment via context-engine clone + `generate-claude-md.py`.
