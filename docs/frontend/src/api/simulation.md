# `frontend/src/api/simulation.js` — Simulation API Wrapper

## Overview

`api/simulation.js` is the largest frontend API wrapper because it covers the whole simulation lifecycle:
- create simulation
- prepare environment
- poll preparation progress
- fetch profiles and config
- start and stop the run
- monitor runtime state
- fetch posts, actions, and history
- interview agents and close the environment

---

## Setup and Preparation

### `createSimulation(data)`
Creates a simulation record via `/api/simulation/create`.

### `prepareSimulation(data)`
Starts asynchronous environment preparation via `/api/simulation/prepare`.

### `getPrepareStatus(data)`
Polls `/api/simulation/prepare/status` for environment-build progress.

### `getSimulation(simulationId)`
Loads the persisted simulation state.

### `getSimulationProfiles(simulationId, platform='reddit')`
Fetches finished agent profiles.

### `getSimulationProfilesRealtime(simulationId, platform='reddit')`
Fetches partially generated profiles while preparation is still running.

### `getSimulationConfig(simulationId)`
Loads the final simulation config.

### `getSimulationConfigRealtime(simulationId)`
Loads config data while the config generator is still running.

### `listSimulations(projectId)`
Lists simulations, optionally filtered by project.

---

## Runtime Control

### `startSimulation(data)`
Starts the live OASIS run.

### `stopSimulation(data)`
Stops a running simulation.

### `getRunStatus(simulationId)`
Loads summary runtime status.

### `getRunStatusDetail(simulationId)`
Loads detailed runtime status including recent actions.

---

## Runtime Data Access

### `getSimulationPosts(simulationId, platform='reddit', limit=50, offset=0)`
Fetches generated platform posts.

### `getSimulationTimeline(simulationId, startRound=0, endRound=null)`
Fetches round-based timeline summaries.

### `getAgentStats(simulationId)`
Loads aggregated agent statistics.

### `getSimulationActions(simulationId, params={})`
Loads action history with optional filters.

---

## Environment and Interaction Helpers

### `closeSimulationEnv(data)`
Requests graceful environment shutdown.

### `getEnvStatus(data)`
Checks whether the OASIS environment is still alive.

### `interviewAgents(data)`
Sends batch interview requests to the backend.

### `getSimulationHistory(limit=20)`
Loads history cards for the home screen.

---

## Related Files

- [`components/Step2EnvSetup.vue`](../components/Step2EnvSetup.md)
- [`components/Step3Simulation.vue`](../components/Step3Simulation.md)
- [`components/HistoryDatabase.vue`](../components/HistoryDatabase.md)
- [`components/Step5Interaction.vue`](../components/Step5Interaction.md)
