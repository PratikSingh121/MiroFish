# `frontend/src/components/HistoryDatabase.vue` — History Browser

## Overview

`HistoryDatabase.vue` shows previous simulations as a grid of cards on the home page. It lets the user reopen a run at the most appropriate workflow step depending on whether the graph, simulation, or report already exists.

---

## State and Layout Constants

Refs:
- `projects`
- `loading`
- `isExpanded`
- `hoveringCard`
- `historyContainer`
- `selectedProject`

Layout constants:
- `CARDS_PER_ROW`
- `CARD_WIDTH`
- `CARD_HEIGHT`
- `CARD_GAP`

Computed:
- `containerStyle`

---

## Functions

### Visual/layout helpers
- `getCardStyle(index)` computes staggered card positioning
- `getProgressClass(simulation)` returns progress styling by simulation stage
- `formatDate(dateStr)` and `formatTime(dateStr)` format timestamps
- `truncateText(text, maxLength)` shortens descriptions
- `getSimulationTitle(requirement)` derives a readable card title
- `formatSimulationId(simulationId)` shortens internal IDs
- `formatRounds(simulation)` formats run-length information
- `getFileType(filename)` and `getFileTypeLabel(filename)` classify uploaded files
- `truncateFilename(filename, maxLength)` shortens file names

### Navigation helpers
- `navigateToProject(simulation)` opens the detail modal for one history card
- `closeModal()` closes that modal
- `goToProject()` routes to graph-build step
- `goToSimulation()` routes to simulation setup or run step
- `goToReport()` routes to the report screen

### Data and observation
- `loadHistory()` loads historical simulations from the backend
- `initObserver()` initializes scroll/visibility behavior for card animation or lazy effects

---

## Lifecycle Role

This component is the frontend entry point for “resume previous work” behavior, so it makes the application feel stateful across sessions instead of single-run only.

---

## Related Files

- [`api/simulation.js`](../api/simulation.md)
- [`views/Home.vue`](../views/Home.md)
