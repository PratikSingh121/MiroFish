# `frontend/src/components/GraphPanel.vue` — Graph Visualization Panel

## Overview

`GraphPanel.vue` is the reusable D3-based graph viewer used across the workflow. It renders the knowledge graph, handles node and edge selection, formats metadata for the detail panel, and exposes refresh and maximize events back to its parent view.

It also contains simulation-aware UI behavior, such as hints shown after a live simulation finishes.

---

## Props and Emits

### Props
- graph data payload
- loading state
- current workflow phase
- whether the app is currently simulating

### Emits
- `refresh`
- `toggle-maximize`

---

## State and Computed Values

Main refs:
- graph container and SVG refs
- `selectedItem`
- `showEdgeLabels`
- `expandedSelfLoops`
- `showSimulationFinishedHint`
- `wasSimulating`

Computed:
- `entityTypes`

---

## Functions

### `dismissFinishedHint()`
Hides the post-simulation hint banner.

### `toggleSelfLoop(id)`
Expands or collapses grouped self-loop edges in the detail UI.

### `formatDateTime(dateStr)`
Formats temporal graph metadata.

### `closeDetailPanel()`
Clears the selected node/edge.

### `renderGraph()`
The core D3 rendering routine.

Responsibilities include:
- mapping backend graph data to D3 nodes and links
- force simulation setup
- node, edge, and label drawing
- click handlers for details
- self-loop and multi-edge visualization logic
- zooming and dragging

### `handleResize()`
Re-renders or adjusts the graph on container resize.

---

## Why This Component Matters

Nearly every route embeds this component, so it functions as the shared visual anchor of the application. The rest of the workflow changes, but the graph remains the common reference surface.

---

## Related Files

- [`views/MainView.vue`](../views/MainView.md)
- [`views/SimulationView.vue`](../views/SimulationView.md)
- [`views/SimulationRunView.vue`](../views/SimulationRunView.md)
- [`views/ReportView.vue`](../views/ReportView.md)
- [`views/InteractionView.vue`](../views/InteractionView.md)
