# `frontend/src/components/Step3Simulation.vue` — Step 3 Workbench

## Overview

`Step3Simulation.vue` manages the live simulation run. It starts and stops the run, polls runtime status and recent actions, displays per-platform progress, and triggers report generation when the user advances to the next step.

This component is the frontend counterpart of [`services/simulation_runner.py`](../../../backend/app/services/simulation_runner.md).

---

## State and Computed Values

Main refs:
- `isGeneratingReport`
- `phase`
- `isStarting`, `isStopping`
- `startError`
- `runStatus`
- `allActions`
- `actionIds` for deduplication
- scroll and log refs
- previous per-platform round refs

Computed:
- `chronologicalActions`
- `twitterActionsCount`
- `redditActionsCount`
- `twitterElapsedTime`
- `redditElapsedTime`

---

## Functions

### `formatElapsedTime(currentRound)`
Converts rounds into human-readable simulated time.

### `addLog(msg)`
Emits timestamped logs upward.

### `resetAllState()`
Clears runtime UI state before a new run.

### `doStartSimulation()`
Calls the start API, resets state, and begins polling.

### `handleStopSimulation()`
Stops the live run.

### Polling control
- `startStatusPolling()`
- `startDetailPolling()`
- `stopPolling()`

### Runtime readers
- `fetchRunStatus()` updates summary status and detects platform completion
- `checkPlatformsCompleted(data)` determines whether both platforms have finished
- `fetchRunStatusDetail()` loads and merges recent actions without duplicates

### Formatting helpers
- `getActionTypeLabel(type)`
- `getActionTypeClass(type)`
- `truncateContent(content, maxLength=100)`
- `formatActionTime(timestamp)`

### `handleNextStep()`
Starts report generation and routes to step 4.

---

## Related Files

- [`api/simulation.js`](../api/simulation.md)
- [`api/report.js`](../api/report.md)
- [`views/SimulationRunView.vue`](../views/SimulationRunView.md)
- [`components/Step4Report.vue`](Step4Report.md)
