# Plot Styling

All plots must have **bold black borders**.

```python
# Matplotlib
for spine in ax.spines.values():
    spine.set_linewidth(2)
    spine.set_color("black")

# Plotly
fig.update_layout(
    xaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
    yaxis=dict(showline=True, linewidth=2, linecolor='black', mirror=True),
)
```
