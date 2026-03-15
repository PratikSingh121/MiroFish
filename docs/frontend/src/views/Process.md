# `frontend/src/views/Process.vue` — Legacy Process Screen

## Overview

`Process.vue` is an older all-in-one implementation of the graph-build screen. It predates the cleaner split into [`MainView.vue`](MainView.md) plus reusable components such as [`GraphPanel.vue`](../components/GraphPanel.md).

Even though the current router points `/process/:projectId` to `MainView.vue`, this file remains useful as a reference for the earlier integrated design.

---

## Responsibilities

This component combines in one file:
- project initialization and pending-upload consumption
- ontology and graph-build status management
- graph polling
- direct D3 rendering and detail-panel logic
- phase tracking and navigation to the next step

---

## Key Functions

### Navigation and UI helpers
- `goHome()` routes back to the landing page
- `goToNextStep()` advances to the next workflow screen
- `toggleFullScreen()` changes graph display mode
- `closeDetailPanel()` closes selected-node/edge details
- `formatDate(dateStr)` formats temporal fields
- `getPhaseStatusClass(phase)` and `getPhaseStatusText(phase)` compute phase badges

### Graph selection and rendering
- `selectNode(nodeData, color)` opens node detail state
- `selectEdge(edgeData)` opens edge detail state
- `renderGraph()` renders the graph directly with D3

### Project and task workflow
- `initProject()`
- `handleNewProject()`
- `loadProject()`
- `updatePhaseByStatus(status)`
- `startBuildGraph()`
- `startGraphPolling()`
- `refreshGraph()`
- `stopGraphPolling()`
- `fetchGraphData()`
- `startPollingTask(taskId)`
- `pollTaskStatus(taskId)`
- `stopPolling()`
- `loadGraph(graphId)`

---

## Why It Matters

This file documents an earlier architecture where page, graph renderer, and workflow logic all lived together. Reading it helps explain why the current codebase extracted separate components for graph display and workflow steps.

---

## Related Files

- [`views/MainView.vue`](MainView.md) is the modern replacement
- [`components/GraphPanel.vue`](../components/GraphPanel.md) replaces the embedded D3 renderer
