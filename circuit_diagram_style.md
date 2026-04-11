# Circuit Diagram Quality Rules

Applies to every schematic you render — CircuiTikZ, TikZ, matplotlib sketches,
napkin drawings. Violations of these rules are what caused the early-iteration
drawings of the V4 JTWPA overview to look "confusing and broken" even after
they compiled cleanly.

Canonical generator: `crucible/src/crucible/viz/circuit.py` (Python →
CircuiTikZ). These rules are enforced in that module; read it as a worked
example before producing new diagrams.

## 1. The drawing must match what is simulated, not the mental model

If the netlist pushes `Cg − Cc` at an RPM node, the schematic shows `Cg − Cc`
at that node. Do NOT "clean up" to the algebraically-equivalent `Cg + Cc·series`
form just because it reads more intuitively. If the convention looks surprising,
add a **footnote** explaining why it's equivalent and why the simulator uses it
— never alter the drawn topology.

**Why:** a schematic's only job is to document what the solver sees. A prettier
but non-matching drawing becomes a trap for the next reader.

## 2. Every electrical node must be connected by a visible wire

Don't rely on coordinate proximity to imply a wire. If two components share a
node but their pins don't literally touch on the page, emit an explicit
`\draw (a) -- (b);` between them. This is the rule that fixed the "floating
`L_j` left pins" bug — cells had a 0.6-unit coordinate gap to their neighbors
and a reader's eye read the chain as broken.

**Concrete pattern:** when a cell repeats across a horizontal step `CELL_DX`
larger than the cell's internal width, include a leading wire segment inside
the cell function that bridges the gap back to the previous element's node.

## 3. Block-diagram boxes must have explicit lead wires in AND out

A gray package block with no wires touching it reads as a floating annotation,
not a component. Every block gets:
- An incoming `\draw` from the previous node to the block's left edge at y=0
- An outgoing `\draw` from the block's right edge to the next node at y=0

The block interior stays empty (the box represents "signal processing happens
here"); the leads on each side are what make it a circuit element.

## 4. When showing a repeating chain, draw at least one representative period

A dashed ellipsis alone tells the reader "something is here" but not what. For
a 1956-cell JTWPA chain with RPMs every 14th cell, the correct treatment is:

- Draw 2–3 full LL cells at the input side
- Draw 2–3 full LL cells at the output side
- Draw a **dashed bus line** between them for the omitted interior
- Draw **one representative RPM stub** rising from the middle of the dashed
  section, fully labeled, so the reader can see what periodic feature is
  being skipped
- Add a brace above everything with the count (e.g. "≈1956 cells, ~139 RPMs
  every 14th cell")

Without the representative stub, the diagram fails to communicate that
periodic RPMs exist at all.

## 5. Vertical stacking for labels — no element touches another element

When annotations, braces, stubs, and footnotes coexist, stack them on distinct
y-coordinates:

- Main chain at y=0
- Shunt ground symbols below at y ≈ −1.7
- Representative stub above at y ∈ [+0.15, +2.5]
- Count brace above the stub at y ≈ +3.1
- Brace label text at y ≈ +3.7

A label that overlaps another element (even partially) is a render bug.
**Always re-render and visually inspect** — don't trust that it "looks OK in
the tex source".

## 6. Horizontal breathing room around labeled components

Leave ≥ 0.5 units of horizontal space between the edge of a component's
bounding box (including its label) and the next component. CircuiTikZ labels
extend the visual footprint well beyond the drawn symbol; tight coordinate
math that ignores labels produces collisions (e.g. `C_g/2` labels clipping
into the output package block).

**Practical minimum:** add ≥ 0.6 units on each side of every labeled shunt
capacitor; more before/after package blocks and port resistors.

## 7. Match the page size to the content — then add margin

CircuiTikZ doesn't auto-crop to content when using `article` + `geometry`
paperwidth/paperheight. Measure the rendered bounding box and pad:

- `paperwidth_mm = x_end * 15 + 20` (rough mm-per-unit scale factor)
- `paperheight_mm ≥ (y_max − y_min) * 15 + 30`

When you add a feature that grows vertically (RPM stub), bump `paperheight`
or the top of the figure gets clipped.

## 8. Footnote the non-obvious

If the drawing faithfully reflects a convention that a naive reader would
flag as "wrong" (the `Cg − Cc` case), add a **scriptsize footnote** below the
main figure explaining:
- What form the drawing shows
- Why it's algebraically equivalent to the naive form
- Which forbidden form it is NOT (e.g. the `Cg − 3·Cc` parallel-branch form
  that breaks unitarity)

A footnote costs two lines and prevents every future reader from re-opening
the same discussion.

## 9. Font hierarchy

- Main circuit element labels: default `\small`
- Annotation boxes (resonance numbers, parameter summaries): `\footnotesize`
- Footnotes explaining topology conventions: `\scriptsize`
- Section headers inside boxes: `\textbf` + same size as box text

Never use `\mathrm` outside math mode. Wrap unit tags in `$...$`.

## 10. Always re-render and inspect visually

Every edit to a circuit renderer must be followed by:
1. `python -c "from crucible.viz import draw_…; draw_…(...)"`
2. Read the PNG with the Read tool (or open in Preview)
3. Check each of rules 1–9 against the rendered image

A green `pdflatex` run only proves the source parsed — it does NOT prove the
output is readable. Rules 5 (label collision) and 7 (page cropping) can only
be verified by looking at the pixels.

## Failure cases this file exists to prevent

- **W14 V4 overview v1:** missing lead wires between cells and between package
  blocks and chain → looked like a set of disconnected components with
  floating labels
- **W14 V4 overview v1 RPM:** no representative stub in the dashed middle →
  reader couldn't tell the chain had periodic RPMs at all
- **W14 V4 RPM cell v1:** self-critique mistakenly flagged the correct
  `Cg − Cc` form as the forbidden `Cg − 3·Cc` form → nearly introduced a
  drawing/netlist mismatch. Fixed by adding rule 1 + a footnote per rule 8
- **W14 V4 overview v2:** brace + cell-count label overlapped the RPM stub
  because all three were stacked at the same y → fixed by rule 5

## Generator: `crucible.viz.circuit`

Public API:
- `draw_rpm_cell(device, workdir, basename)` — single-cell zoom (rule 1
  compliant: uses the `Cg − Cc` form + footnote per rule 8)
- `draw_parasitic_network(params, workdir, basename, side)` — input-side
  package launch (L_ard + PCB strip + C_bump + Z-taper ladder)
- `draw_jtwpa_overview(device, workdir, parasitics, basename)` — full-chain
  block diagram (representative RPM stub per rule 4; package leads per rule 3)

All three return a `Rendered` dataclass with `.tex_path`, `.pdf_path`, and
`.png_paths`. Inspect the PNG before claiming the drawing is done.
