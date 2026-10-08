---
name: data-visualization
description: Chart and plot data — pick the right chart type for the data shape, then generate Chart.js config or Plotly code. Use when asked to visualize a dataset, plot results, build a chart, or choose between chart types/libraries for quantitative data (as opposed to structural/technical diagrams — see technical-diagrams for those).
---

# Data Visualization

For charting *quantitative data* (numbers, series, distributions). For structural diagrams (architecture, flowcharts, schemas), use `technical-diagrams` instead — this skill is about plotting data, not drawing systems.

## Choosing a chart type

| Data shape | Chart | Avoid |
|---|---|---|
| Trend over time | Line chart | Bar chart (implies discrete categories) |
| Comparing categories | Bar chart (horizontal if labels are long) | Pie chart beyond ~5 slices |
| Part-to-whole, few categories | Pie/donut | Pie chart with >5-6 slices — switch to a bar chart |
| Distribution of one variable | Histogram or box plot | Line chart |
| Relationship between two continuous variables | Scatter plot | Bar chart |
| Relationship between two continuous variables, many points, need density | Hexbin or 2D density plot | Scatter plot (overplots) |
| Multiple series compared over time | Multi-line chart, or small multiples if series count > ~5 | Stacked area (hides individual trends) |
| Hierarchical part-to-whole | Treemap or sunburst | Nested pie charts |
| Matrix/correlation data | Heatmap | Table of numbers |

Default to the simplest chart that shows the comparison you actually care about. A chart that needs a paragraph to explain is usually the wrong chart.

## Chart.js

Good for interactive web dashboards and lightweight embeds. Minimal config shape:
```js
new Chart(ctx, {
  type: 'line', // 'bar' | 'pie' | 'doughnut' | 'scatter' | 'radar'
  data: {
    labels: ['Jan', 'Feb', 'Mar'],
    datasets: [{
      label: 'Series A',
      data: [12, 19, 3],
      borderColor: '#2563EB',
      backgroundColor: 'rgba(37, 99, 235, 0.1)',
      tension: 0.3, // smooths line charts
    }],
  },
  options: {
    responsive: true,
    scales: { y: { beginAtZero: true } },
    plugins: { legend: { position: 'top' } },
  },
});
```
Use for: dashboards, README-embedded charts, anywhere the chart needs to live inside a web page with interactivity (tooltips, zoom) but doesn't need heavy statistical plot types.

## Plotly

Better for exploratory/scientific plotting — more chart types out of the box (contour, 3D surface, box plots with statistical annotations), and works the same in Python, JS, and R. Python example:
```python
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Scatter(x=x, y=y, mode='lines+markers', name='Series A'))
fig.update_layout(
    title='Result',
    xaxis_title='Epoch',
    yaxis_title='Loss',
    template='plotly_white',
)
fig.show()  # or fig.write_html('out.html') / fig.write_image('out.png')
```
Use for: research figures (loss curves, ablations, distributions), anything needing 3D or contour plots, or when the same plotting code should work in a Jupyter notebook and a script without adaptation.

**For static publication figures** (papers, not dashboards): prefer `matplotlib`/`seaborn` in Python over either of the above — they give tighter control over publication-style output (font embedding, exact DPI, vector PDF/EPS export) that Chart.js and Plotly's raster/HTML-first defaults don't match as cleanly.

## General practice

- Label axes and units always; a chart without axis labels isn't finished.
- Pick a colorblind-safe, perceptually uniform palette for anything with >2 series or a continuous scale (e.g. Viridis for heatmaps/continuous data, not rainbow/jet).
- Show uncertainty where it exists — error bars, confidence bands, or a box plot instead of just the mean.
- Don't truncate a bar chart's y-axis at a non-zero value; it exaggerates differences. Line charts showing a trend (not a magnitude comparison) are the one common exception.
