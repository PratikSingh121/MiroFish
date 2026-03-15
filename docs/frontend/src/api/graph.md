# `frontend/src/api/graph.js` — Graph API Wrapper

## Overview

`api/graph.js` wraps all frontend requests related to project creation, ontology generation, graph building, task polling, and graph retrieval.

Every exported function is a thin wrapper around the shared Axios client in [`index.js`](index.md).

---

## Functions

### `generateOntology(formData)`

Posts multipart form data to `/api/graph/ontology/generate`.

Used when the user uploads documents and a simulation requirement from the home screen.

### `buildGraph(data)`

Posts to `/api/graph/build` to start graph construction after ontology generation succeeds.

### `getTaskStatus(taskId)`

GETs `/api/graph/task/<taskId>` to poll long-running task progress.

### `getGraphData(graphId)`

GETs `/api/graph/data/<graphId>` and returns the graph payload used by the D3 panel.

### `getProject(projectId)`

GETs `/api/graph/project/<projectId>` to load project metadata and status.

---

## Related Files

- [`views/Home.vue`](../views/Home.md) starts ontology generation indirectly
- [`views/MainView.vue`](../views/MainView.md) drives graph build and task polling
- [`components/GraphPanel.vue`](../components/GraphPanel.md) consumes graph data
