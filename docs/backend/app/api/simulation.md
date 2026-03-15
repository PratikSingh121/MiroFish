# `backend/app/api/simulation.py` — Simulation API Routes

## Overview

`simulation.py` defines all HTTP endpoints under the `/api/simulation` prefix. It covers the **entire lifecycle** of an OASIS simulation: creating a simulation instance, preparing agent profiles and configuration, running the simulation, monitoring it in real-time, interacting with agents, and downloading results.

This file is the most complex API module — it coordinates between `ZepEntityReader`, `OasisProfileGenerator`, `SimulationManager`, `SimulationRunner`, and `SimulationIPCClient`.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `ZepEntityReader` | [`services/zep_entity_reader.py`](../services/zep_entity_reader.md) | Fetch entities from Zep graph |
| `OasisProfileGenerator` | [`services/oasis_profile_generator.py`](../services/oasis_profile_generator.md) | Generate agent personas |
| `SimulationManager`, `SimulationStatus` | [`services/simulation_manager.py`](../services/simulation_manager.md) | Create/manage simulation state |
| `SimulationRunner`, `RunnerStatus` | [`services/simulation_runner.py`](../services/simulation_runner.md) | Subprocess runner for OASIS |
| `ProjectManager` | [`models/project.py`](../models/project.md) | Retrieve project data |
| `TaskManager`, `TaskStatus` | [`models/task.py`](../models/task.md) | Track async prepare tasks |
| `Config` | [`config.py`](../config.md) | `ZEP_API_KEY`, `OASIS_SIMULATION_DATA_DIR` |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Constants

### `INTERVIEW_PROMPT_PREFIX`

```python
INTERVIEW_PROMPT_PREFIX = "结合你的人设、所有的过往记忆与行动，不调用任何工具直接用文本回复我："
```

A prefix appended to interview prompts to instruct the LLM agent to respond with plain text rather than calling Zep tools. This avoids unwanted tool invocations during "interview" interactions.

---

## Helper Functions

### `optimize_interview_prompt(prompt)`

```python
def optimize_interview_prompt(prompt: str) -> str:
```

Prepends `INTERVIEW_PROMPT_PREFIX` to `prompt` only if it is not already present. Used before sending interview commands to agents via the IPC system.

---

### `_check_simulation_prepared(simulation_id)`

```python
def _check_simulation_prepared(simulation_id: str) -> tuple[bool, dict]:
```

**Why it exists:** The preparation step (generating profiles and config) is expensive and should not be repeated. This function checks the simulation directory for evidence of a completed preparation:

1. Verifies the simulation directory exists.
2. Checks that all required files are present: `state.json`, `simulation_config.json`, `reddit_profiles.json`, `twitter_profiles.csv`.
3. Reads `state.json` and checks `status` and `config_generated`.
4. If status is `"preparing"` but all files are present and `config_generated=True`, it **auto-upgrades** the status to `"ready"` and rewrites `state.json`.

**Returns:** `(is_prepared: bool, info: dict)` — info contains profile counts, entity types, file list, timestamps.

---

## Entity Endpoints

### `GET /api/simulation/entities/<graph_id>`
**`get_graph_entities(graph_id)`**

Returns all entities from the Zep graph that have a non-generic label (i.e. typed entities, not plain `Entity` nodes).

Query params:
- `entity_types` — comma-separated list to filter by type
- `enrich` — `true`/`false` (default `true`) — whether to include related edges

Uses `ZepEntityReader.filter_defined_entities()`.

---

### `GET /api/simulation/entities/<graph_id>/<entity_uuid>`
**`get_entity_detail(graph_id, entity_uuid)`**

Returns details for a single entity including its related edges and neighbouring nodes. Uses `ZepEntityReader.get_entity_with_context()`.

---

### `GET /api/simulation/entities/<graph_id>/by-type/<entity_type>`
**`get_entities_by_type(graph_id, entity_type)`**

Returns all entities of a specific type. Uses `ZepEntityReader.get_entities_by_type()`.

---

## Simulation Lifecycle Endpoints

### `POST /api/simulation/create`
**`create_simulation()`**

Creates a new `SimulationState` and saves it to disk. Requires `project_id` in the JSON body; optionally accepts `graph_id`, `enable_twitter`, `enable_reddit`.

Returns the `SimulationState` dict including the newly-assigned `simulation_id`.

---

### `POST /api/simulation/prepare`
**`prepare_simulation()`**

**Most complex endpoint.** Prepares all files needed to run OASIS.

**Smart idempotency:** First calls `_check_simulation_prepared()`. If already prepared and `force_regenerate=false`, returns immediately without doing any work.

**Pre-flight entity count:** Before launching the background thread, synchronously calls `ZepEntityReader.filter_defined_entities()` with `enrich_with_edges=False` to quickly count entities. This count is immediately saved to `SimulationState` so the frontend can display a "expected agent total" number right away.

**Background task steps:**
1. Filter entities (`ZepEntityReader.filter_defined_entities()`)
2. Generate agent profiles (`OasisProfileGenerator.generate_profiles()`) in parallel batches
3. Generate simulation config (`SimulationConfigGenerator.generate()`)
4. Save `reddit_profiles.json`, `twitter_profiles.csv`, `simulation_config.json`
5. Update `state.json` with `status="ready"` and `config_generated=True`

Returns `task_id` immediately.

---

### `POST /api/simulation/prepare/status`
**`get_prepare_status()`**

Polls the status of a running prepare task. Accepts `task_id` or `simulation_id` in the JSON body. If `simulation_id` is given, also checks directly in the filesystem via `_check_simulation_prepared()`.

---

### `GET /api/simulation/<simulation_id>`
**`get_simulation(simulation_id)`**

Returns the current `SimulationState.to_dict()` for the given simulation.

---

### `GET /api/simulation/<simulation_id>/profiles`
**`get_simulation_profiles(simulation_id)`**

Returns the saved agent profiles from `reddit_profiles.json` or `twitter_profiles.csv` depending on the `platform` query parameter.

---

### `GET /api/simulation/<simulation_id>/profiles/realtime`
**`get_simulation_profiles_realtime(simulation_id)`**

Returns profiles as they are being generated (reads whatever is written to disk so far). Used for live progress display in the frontend.

---

### `GET /api/simulation/<simulation_id>/config`
**`get_simulation_config(simulation_id)`**

Returns the complete `simulation_config.json` for the given simulation.

---

### `POST /api/simulation/start`
**`start_simulation()`**

Launches the OASIS simulation as a background subprocess via `SimulationRunner.start()`.

Body: `{ "simulation_id": "...", "enable_graph_memory_update": true }`

The `enable_graph_memory_update` flag controls whether agent actions are fed back into the Zep graph in real-time via `ZepGraphMemoryManager`.

---

### `POST /api/simulation/stop`
**`stop_simulation()`**

Sends a stop signal to the running simulation subprocess via `SimulationRunner.stop()`.

---

### `GET /api/simulation/<simulation_id>/run-status`
**`get_run_status(simulation_id)`**

Returns a summary of the current simulation run: current round, total rounds, actions per platform, platform running/completed booleans.

---

### `GET /api/simulation/<simulation_id>/run-status/detail`
**`get_run_status_detail(simulation_id)`**

Same as above but also includes the last 50 agent actions for live feed display.

---

### `GET /api/simulation/<simulation_id>/posts`
**`get_simulation_posts(simulation_id)`**

Returns posts from the OASIS action log for a given platform, with pagination.

---

### `GET /api/simulation/<simulation_id>/timeline`
**`get_simulation_timeline(simulation_id)`**

Returns per-round summaries between `start_round` and `end_round`.

---

### `GET /api/simulation/<simulation_id>/agent-stats`
**`get_agent_stats(simulation_id)`**

Returns per-agent statistics: total posts, total actions broken down by type.

---

### Interaction / Interview Endpoints

#### `POST /api/simulation/<simulation_id>/interview`
**`interview_agent(simulation_id)`**

Sends a question to a specific agent via the IPC system (`SimulationIPCClient.send_interview()`). The agent's LLM responds in-character using its Zep memory. The `optimize_interview_prompt()` helper ensures the agent replies with text instead of tool calls.

#### `POST /api/simulation/<simulation_id>/broadcast`
**`broadcast_to_agents(simulation_id)`**

Sends a message to all agents simultaneously (batch interview).

---

## Related Files

- [`services/zep_entity_reader.py`](../services/zep_entity_reader.md)
- [`services/oasis_profile_generator.py`](../services/oasis_profile_generator.md)
- [`services/simulation_manager.py`](../services/simulation_manager.md)
- [`services/simulation_runner.py`](../services/simulation_runner.md)
- [`services/simulation_ipc.py`](../services/simulation_ipc.md)
- [`models/project.py`](../models/project.md)
- [`models/task.py`](../models/task.md)
- Frontend: [`frontend/src/api/simulation.js`](../../../frontend/src/api/simulation.md)
