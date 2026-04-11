# Plot Styling — DARK THEME (permanent preference)

**Always use dark-theme plots** unless I explicitly ask for light theme or publication-ready. This applies to every plot you make, in every project, with every tool (Plotly, Matplotlib, Vega-Lite, etc.). Subagents you spawn must follow this rule too — pass it forward in prompts.

## Required properties

- **Background**: near-black `#0a0a0a` on both paper and plot area.
- **Text**: bright off-white `#f0f0f0`, minimum 12 px.
- **Title**: bold, bright white `#ffffff`, size ≥ 16.
- **Axis frame**: *bold, thick, mirrored*, bright white, `linewidth ≥ 3`.
- **Axis ticks**: outside, bright white, `tickwidth ≥ 2`.
- **Gridlines**: subtle dark gray `rgba(100,100,100,0.25)`, thin (1 px).
- **Zero line**: off (clutter).
- **Legend**: semi-transparent dark bg `rgba(30,30,30,0.85)`, bright border, bright text.
- **Trace colors**: saturated and high-contrast; use the palette below.
- **Line width**: ≥ 2 px for primary traces; 1–1.5 px for reference/dashed overlays.

## Palette

```python
DARK_PALETTE = ['#ff453a', '#ffd60a', '#32d74b', '#0a84ff', '#bf5af2',
                '#64d2ff', '#ff9f0a', '#ff2d55', '#30d158', '#5e5ce6',
                '#ffb340', '#ac8e68']
```

Red / yellow / green / blue / purple / cyan / orange / pink / mint / indigo / amber / tan. These pop against `#0a0a0a` and remain distinguishable when the plot is downsized for slides.

## Plotly snippet (copy-paste)

```python
DARK = dict(
    paper_bgcolor='#0a0a0a', plot_bgcolor='#0a0a0a',
    font=dict(color='#f0f0f0', size=13, family='Arial, sans-serif'),
    title=dict(font=dict(color='#ffffff', size=16, family='Arial Black')),
    xaxis=dict(linecolor='#ffffff', linewidth=3, mirror=True, showline=True,
               ticks='outside', tickcolor='#ffffff', tickwidth=2,
               tickfont=dict(color='#f0f0f0'),
               gridcolor='rgba(100,100,100,0.25)', gridwidth=1, zeroline=False),
    yaxis=dict(linecolor='#ffffff', linewidth=3, mirror=True, showline=True,
               ticks='outside', tickcolor='#ffffff', tickwidth=2,
               tickfont=dict(color='#f0f0f0'),
               gridcolor='rgba(100,100,100,0.25)', gridwidth=1, zeroline=False),
    legend=dict(bgcolor='rgba(30,30,30,0.85)', bordercolor='#ffffff',
                borderwidth=1, font=dict(color='#f0f0f0', size=11)),
)
DARK_PALETTE = ['#ff453a', '#ffd60a', '#32d74b', '#0a84ff', '#bf5af2',
                '#64d2ff', '#ff9f0a', '#ff2d55', '#30d158', '#5e5ce6',
                '#ffb340', '#ac8e68']
# For multi-axis layouts, replicate the xaxis/yaxis dict under xaxis2, yaxis2, etc.
```

## Matplotlib snippet

```python
import matplotlib.pyplot as plt
plt.style.use('dark_background')
plt.rcParams.update({
    'axes.linewidth': 2.5, 'axes.edgecolor': '#ffffff',
    'xtick.major.width': 2, 'ytick.major.width': 2,
    'xtick.color': '#f0f0f0', 'ytick.color': '#f0f0f0',
    'axes.labelcolor': '#f0f0f0', 'axes.titlecolor': '#ffffff',
    'axes.labelsize': 13, 'axes.titlesize': 15, 'axes.titleweight': 'bold',
    'figure.facecolor': '#0a0a0a', 'axes.facecolor': '#0a0a0a',
    'savefig.facecolor': '#0a0a0a',
    'grid.color': '#646464', 'grid.alpha': 0.25, 'grid.linewidth': 1,
    'legend.facecolor': '#1e1e1e', 'legend.edgecolor': '#ffffff',
    'legend.framealpha': 0.85,
})
```

## Rationale (so future agents understand, not just follow)

- **Dark background**: reduces eye strain during long sim-analysis sessions.
- **Bold, thick frames**: survive downscaling to slides / paper figures; a thin frame disappears at 50% zoom.
- **Bright saturated colors**: disambiguate dense overlays (frequently 8-12 traces) against a neutral dark field.
- **Off-white (`#f0f0f0`), not pure white**: pure `#ffffff` text on `#0a0a0a` is too high contrast and causes afterimages.

## Migration

This file lives at `~/projects/twpa/.claude/plot_styling.md` and auto-syncs on every edit to:
1. `github.ibm.com/Ruichen-Zhao/claude_notebook` (origin)
2. `github.com/ruichen77/claude-notebook` (github)

To bring these preferences to a new machine, clone either repo into `~/projects/twpa/.claude/` and Claude Code picks them up automatically.

## When to deviate

Only if I explicitly say "light theme", "publication-ready", or "matches paper X's style" for a specific plot. Default is always dark.
