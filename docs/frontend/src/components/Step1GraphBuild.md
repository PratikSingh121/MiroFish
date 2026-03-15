# `frontend/src/components/Step1GraphBuild.vue` — Step 1 Workbench

## Overview

`Step1GraphBuild.vue` is the right-hand workbench for the graph-build step. It presents ontology generation results, build progress, graph statistics, and the control that creates a simulation record and advances into environment setup.

---

## Props

This component receives the current phase, project data, ontology/build progress objects, graph data, and the shared system log stream from its parent.

---

## State and Computed Values

Refs:
- `selectedOntologyItem`
- `logContent`
- `creatingSimulation`

Computed:
- `graphStats`

---

## Functions

### `handleEnterEnvSetup()`
Creates a simulation by calling the backend and then routes into step 2.

### `selectOntologyItem(item, type)`
Stores which entity or edge type the user clicked so details can be shown.

### `formatDate(dateStr)`
Formats timestamps for the UI.

---

## Related Files

- [`api/simulation.js`](../api/simulation.md)
- [`views/MainView.vue`](../views/MainView.md)
- [`components/Step2EnvSetup.vue`](Step2EnvSetup.md)
