# `frontend/src/views/SimulationRunView.vue` — Step 3: Simulation Run

## Overview

`SimulationRunView.vue` is the route-level shell for the live simulation step. It keeps the graph panel visible, loads timing metadata such as `minutes_per_round`, and hosts [`Step3Simulation.vue`](../components/Step3Simulation.md), which handles the actual runtime monitoring and controls.

---

## State and Computed Values

Main refs:
- `viewMode`
- `currentSimulationId`
- `maxRounds` from the route query
- `minutesPerRound`
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
- `isSimulating`

---

## Functions

### `addLog(msg)`
Adds timestamped logs.

### `updateStatus(status)`
Updates the page-level run status.

### `toggleMaximize(target)`
Switches layout mode.

### `handleGoBack()`
Before routing back to step 2, tries to close the live simulation environment or stop the subprocess if needed.

### `handleNextStep()`
Backup handler for moving to report generation. In practice, the child component usually drives the report-generation route transition directly.

### `loadSimulationData()`
Loads simulation metadata, reads the simulation config to get `minutes_per_round`, loads project data, then loads graph data.

### `loadGraph(graphId)`
Loads graph data with reduced spinner behavior while the simulation is running.

### `refreshGraph()`
Manual graph refresh.

### `startGraphRefresh()`
Starts periodic graph refresh during live simulation.

### `stopGraphRefresh()`
Stops that polling loop.

---

## Child Components

- [`components/GraphPanel.vue`](../components/GraphPanel.md)
- [`components/Step3Simulation.vue`](../components/Step3Simulation.md)

---

## Related Files

- [`api/simulation.js`](../api/simulation.md)
- [`views/ReportView.vue`](ReportView.md)
