# `frontend/src/views/InteractionView.vue` — Step 5: Deep Interaction

## Overview

`InteractionView.vue` is the final route shell. It loads the report and associated simulation/project/graph context, keeps the graph panel available, and renders the interactive chat and survey UI through [`Step5Interaction.vue`](../components/Step5Interaction.md).

Unlike earlier steps, this screen defaults to the workbench view because the graph is reference material rather than the main focus.

---

## State and Computed Values

Refs:
- `viewMode`
- `currentReportId`
- `simulationId`
- `projectData`
- `graphData`
- `graphLoading`
- `systemLogs`
- `currentStatus`

Computed:
- `leftPanelStyle`
- `rightPanelStyle`
- `statusClass`
- `statusText`

---

## Functions

### `addLog(msg)`
Adds timestamped logs.

### `updateStatus(status)`
Updates the route-level status badge.

### `toggleMaximize(target)`
Changes layout mode.

### `loadReportData()`
Loads the report, then the linked simulation, then the linked project, then graph data.

### `loadGraph(graphId)`
Fetches graph payload for the graph panel.

### `refreshGraph()`
Manual graph reload.

The component watches `route.params.reportId` and reloads when needed.

---

## Child Components

- [`components/GraphPanel.vue`](../components/GraphPanel.md)
- [`components/Step5Interaction.vue`](../components/Step5Interaction.md)

---

## Related Files

- [`api/report.js`](../api/report.md)
- [`api/simulation.js`](../api/simulation.md)
