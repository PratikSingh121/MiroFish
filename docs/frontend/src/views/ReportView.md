# `frontend/src/views/ReportView.vue` — Step 4: Report Generation

## Overview

`ReportView.vue` is the route shell for report generation. It loads report, simulation, project, and graph context, shows the graph panel on the left, and delegates the reporting UI to [`Step4Report.vue`](../components/Step4Report.md).

The default layout favors the workbench because the report timeline and tool logs are the focus at this stage.

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
Adds timestamped page logs.

### `updateStatus(status)`
Updates the page badge.

### `toggleMaximize(target)`
Changes layout mode.

### `loadReportData()`
Loads the report, discovers its simulation ID, then loads the associated simulation, project, and graph.

### `loadGraph(graphId)`
Loads graph data for the left panel.

### `refreshGraph()`
Manual graph refresh handler.

The route also watches `route.params.reportId` so it can reload if the URL changes.

---

## Child Components

- [`components/GraphPanel.vue`](../components/GraphPanel.md)
- [`components/Step4Report.vue`](../components/Step4Report.md)

---

## Related Files

- [`api/report.js`](../api/report.md)
- [`views/InteractionView.vue`](InteractionView.md)
