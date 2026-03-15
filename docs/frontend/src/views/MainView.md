# `frontend/src/views/MainView.vue` — Step 1: Graph Build

## Overview

`MainView.vue` is the first main workflow screen after the landing page. It manages project initialization, ontology generation follow-through, graph build startup, task polling, graph polling, and the transition into simulation setup.

Its layout is split into:
- a left graph visualization panel
- a right workbench that swaps between step 1 and step 2 components

---

## State

Main groups:
- layout: `viewMode`
- workflow: `currentStep`, `stepNames`, `currentPhase`
- project/graph data: `currentProjectId`, `projectData`, `graphData`
- progress: `ontologyProgress`, `buildProgress`
- UI/error: `loading`, `graphLoading`, `error`, `systemLogs`
- timers: `pollTimer`, `graphPollTimer`

### Computed

- `leftPanelStyle`
- `rightPanelStyle`
- `statusClass`
- `statusText`

---

## Functions

### `addLog(msg)`
Appends timestamped messages to the workbench log list.

### `toggleMaximize(target)`
Switches between graph-only, split, and workbench-only modes.

### `handleNextStep(params = {})`
Moves from step 1 to step 2 inside the same view and records context like custom round counts.

### `handleGoBack()`
Moves back one internal step.

### `initProject()`
Entry point on mount. Chooses between `handleNewProject()` and `loadProject()`.

### `handleNewProject()`
Consumes the pending-upload store, uploads files through `generateOntology()`, clears the store, updates the route, and immediately starts graph building.

### `loadProject()`
Loads an existing project and resumes the correct phase based on project status.

### `updatePhaseByStatus(status)`
Maps backend project states to frontend phase numbers.

### `startBuildGraph()`
Calls the graph-build API and starts both task polling and graph polling.

### `startGraphPolling()`
Begins periodic graph refreshes while the graph is being built.

### `fetchGraphData()`
Reloads project metadata and graph data to keep the left panel current.

### `startPollingTask(taskId)`
Creates the build-task polling loop.

### `pollTaskStatus(taskId)`
Reads task progress, updates phase state, handles completion, and loads the final graph when ready.

### `loadGraph(graphId)`
Loads the full graph payload into the graph panel.

### `refreshGraph()`
Manual graph refresh handler.

### `stopPolling()`
Stops task-status polling.

### `stopGraphPolling()`
Stops periodic graph refresh polling.

---

## Child Components

- [`components/GraphPanel.vue`](../components/GraphPanel.md)
- [`components/Step1GraphBuild.vue`](../components/Step1GraphBuild.md)
- [`components/Step2EnvSetup.vue`](../components/Step2EnvSetup.md)

---

## Related Files

- [`api/graph.js`](../api/graph.md)
- [`store/pendingUpload.js`](../store/pendingUpload.md)
- [`views/SimulationView.vue`](SimulationView.md)
