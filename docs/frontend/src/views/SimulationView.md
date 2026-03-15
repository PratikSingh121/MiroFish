# `frontend/src/views/SimulationView.vue` — Step 2: Environment Setup

## Overview

`SimulationView.vue` is the route-level shell for the environment-setup step. It loads the simulation, project, and graph context, renders the graph on the left, and delegates the actual preparation workflow to [`Step2EnvSetup.vue`](../components/Step2EnvSetup.md).

It also contains defensive logic for users returning from a running simulation: if the OASIS environment is still alive, the view tries to shut it down before resuming setup.

---

## State and Computed Values

Main refs:
- `viewMode`
- `currentSimulationId`
- `projectData`
- `graphData`
- `graphLoading`
- `systemLogs`
- `currentStatus`

Computed helpers:
- `leftPanelStyle`
- `rightPanelStyle`
- `statusClass`
- `statusText`

---

## Functions

### `addLog(msg)`
Adds timestamped UI log entries.

### `updateStatus(status)`
Updates the route-level status badge.

### `toggleMaximize(target)`
Toggles between graph, split, and workbench layouts.

### `handleGoBack()`
Routes back to the process screen for the project.

### `handleNextStep(params = {})`
Navigates to the simulation-run screen and forwards optional `maxRounds` as a query parameter.

### `checkAndStopRunningSimulation()`
Checks whether a previous simulation process is still alive and attempts to close or stop it before setup continues.

### `forceStopSimulation()`
Fallback hard-stop path if graceful shutdown fails.

### `loadSimulationData()`
Loads simulation metadata, then project data, then graph data.

### `loadGraph(graphId)`
Loads graph payload for the left panel.

### `refreshGraph()`
Manual graph refresh.

---

## Child Components

- [`components/GraphPanel.vue`](../components/GraphPanel.md)
- [`components/Step2EnvSetup.vue`](../components/Step2EnvSetup.md)

---

## Related Files

- [`api/simulation.js`](../api/simulation.md)
- [`views/SimulationRunView.vue`](SimulationRunView.md)
