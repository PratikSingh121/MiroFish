# `frontend/src/router/index.js` — Router Configuration

## Overview

`router/index.js` defines the page-level workflow of the frontend. Each route corresponds to one stage of the MiroFish pipeline.

---

## Routes

### `/` → `Home`
Renders [`views/Home.vue`](../views/Home.md).

### `/process/:projectId` → `Process`
Renders [`views/MainView.vue`](../views/MainView.md). This is step 1 of the main workflow.

### `/simulation/:simulationId` → `Simulation`
Renders [`views/SimulationView.vue`](../views/SimulationView.md). This is step 2.

### `/simulation/:simulationId/start` → `SimulationRun`
Renders [`views/SimulationRunView.vue`](../views/SimulationRunView.md). This is step 3.

### `/report/:reportId` → `Report`
Renders [`views/ReportView.vue`](../views/ReportView.md). This is step 4.

### `/interaction/:reportId` → `Interaction`
Renders [`views/InteractionView.vue`](../views/InteractionView.md). This is step 5.

All workflow routes enable `props: true`, allowing route params to be passed as component props.

---

## Router Instance

### `createRouter({ history: createWebHistory(), routes })`

Creates an HTML5-history router with the route table above.

---

## Related Files

- [`main.js`](../main.md) installs the router
- every view file listed above participates in the step-by-step UX
