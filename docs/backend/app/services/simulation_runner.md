# `backend/app/services/simulation_runner.py` — Simulation Runner

## Overview

`simulation_runner.py` is responsible for **launching and monitoring the OASIS simulation subprocess** and providing real-time status updates to the Flask API.

When the API receives a "start" request, `SimulationRunner.start_simulation()` launches the appropriate Python script in `backend/scripts/` as a subprocess, then spawns a monitor thread that tails the JSONL action log files written by OASIS. The monitor thread parses every agent action, updates a `SimulationRunState` object in memory, and saves it to disk so the API can serve polling requests cheaply.

The runner handles both single-platform (Twitter or Reddit) and dual-platform (parallel) launches. It is also responsible for cleanly stopping simulations — with cross-platform process-tree termination support (Windows `taskkill` vs Unix `os.killpg`).

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `ZepGraphMemoryManager` | [`services/zep_graph_memory_updater.py`](zep_graph_memory_updater.md) | Optional Zep updater for live graph updates |
| `SimulationIPCClient`, `CommandType`, `IPCResponse` | [`services/simulation_ipc.py`](simulation_ipc.md) | File-based IPC for interview commands |
| `Config` | [`app/config.py`](../config.md) | API keys |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `IS_WINDOWS` | `sys.platform == 'win32'` | Used to select process-termination strategy |
| `_cleanup_registered` | `False` | Module-level flag preventing duplicate `atexit` registration |

---

## Enums

### `RunnerStatus`

```
IDLE → STARTING → RUNNING → PAUSED → STOPPING → STOPPED / COMPLETED / FAILED
```

| Value | Meaning |
|-------|---------|
| `IDLE` | No simulation has been started yet |
| `STARTING` | `Popen` launched, monitor thread not yet confirmed running |
| `RUNNING` | Monitor thread active, process is live |
| `PAUSED` | Simulation paused (IPC command sent) |
| `STOPPING` | Termination in progress |
| `STOPPED` | Manually stopped |
| `COMPLETED` | Natural completion detected via `simulation_end` event |
| `FAILED` | Process exit code non-zero or monitor thread exception |

---

## Data Structures

### `AgentAction`

A single logged action from one OASIS agent in one round.

| Field | Type | Description |
|-------|------|-------------|
| `round_num` | `int` | Simulation round |
| `timestamp` | `str` | ISO datetime |
| `platform` | `str` | `"twitter"` or `"reddit"` |
| `agent_id` | `int` | Numeric agent ID |
| `agent_name` | `str` | Agent's display name |
| `action_type` | `str` | e.g., `CREATE_POST`, `LIKE_POST`, `FOLLOW_USER` |
| `action_args` | `dict` | Arguments passed to the action |
| `result` | `Optional[str]` | LLM-generated action output / post content |
| `success` | `bool` | Whether the action succeeded |

#### `AgentAction.to_dict()`
Serialises all fields to a plain dict for API responses and file persistence.

---

### `RoundSummary`

Aggregates all actions in one simulation round.

| Field | Type | Description |
|-------|------|-------------|
| `round_num` | `int` | Round number |
| `start_time` / `end_time` | `str` | ISO timestamps |
| `simulated_hour` | `int` | Simulated clock hour for this round |
| `twitter_actions` | `int` | Count of Twitter actions in this round |
| `reddit_actions` | `int` | Count of Reddit actions in this round |
| `active_agents` | `List[int]` | IDs of agents active this round |
| `actions` | `List[AgentAction]` | All actions this round |

---

### `SimulationRunState`

The live, in-memory (and periodically persisted) view of a running simulation.

| Field | Type | Description |
|-------|------|-------------|
| `simulation_id` | `str` | The simulation ID |
| `runner_status` | `RunnerStatus` | Overall status |
| `current_round` | `int` | Max round seen across all platforms |
| `total_rounds` | `int` | Total rounds to complete |
| `simulated_hours` | `int` | Max simulated hours across all platforms |
| `total_simulation_hours` | `int` | Total hours from config |
| `twitter_current_round` / `reddit_current_round` | `int` | Per-platform current round |
| `twitter_simulated_hours` / `reddit_simulated_hours` | `int` | Per-platform simulated time |
| `twitter_running` / `reddit_running` | `bool` | Whether each platform is still live |
| `twitter_completed` / `reddit_completed` | `bool` | Whether each platform has finished |
| `twitter_actions_count` / `reddit_actions_count` | `int` | Cumulative action counts |
| `rounds` | `List[RoundSummary]` | Historical round summaries |
| `recent_actions` | `List[AgentAction]` | Last 50 actions (most recent first) |
| `started_at` / `updated_at` / `completed_at` | `str` | Timestamps |
| `error` | `Optional[str]` | Error message if failed |
| `process_pid` | `Optional[int]` | OS PID of the OASIS subprocess |

#### `SimulationRunState.add_action(action)`

Inserts `action` at the front of `recent_actions`, truncates to 50 entries, and increments the platform action counter.

#### `SimulationRunState.to_dict()` / `to_detail_dict()`

`to_dict()` returns the summary view (no action list). `to_detail_dict()` adds `recent_actions` and `rounds_count`.

---

## `SimulationRunner` Class (all class methods)

`SimulationRunner` is never instantiated — all state is managed in class-level dictionaries:

| Attribute | Type | Description |
|-----------|------|-------------|
| `_run_states` | `Dict[str, SimulationRunState]` | In-memory state cache |
| `_processes` | `Dict[str, subprocess.Popen]` | Running subprocesses |
| `_action_queues` | `Dict[str, Queue]` | Per-simulation action queues |
| `_monitor_threads` | `Dict[str, Thread]` | Background monitor threads |
| `_stdout_files` / `_stderr_files` | `Dict[str, file]` | Open log file handles |
| `_graph_memory_enabled` | `Dict[str, bool]` | Whether Zep graph updates are active |

### `get_run_state(simulation_id) → Optional[SimulationRunState]`

Returns the cached state or loads from `run_state.json` on disk.

---

### `_load_run_state(simulation_id) → Optional[SimulationRunState]`

Reads `uploads/simulations/<sim_id>/run_state.json` and reconstructs a `SimulationRunState` dataclass with full `recent_actions` list.

---

### `_save_run_state(state)`

Writes `state.to_detail_dict()` to `run_state.json` and updates the in-memory cache.

---

### `start_simulation(simulation_id, platform, max_rounds, enable_graph_memory_update, graph_id) → SimulationRunState`

**Starts the OASIS subprocess and monitoring thread.**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `simulation_id` | `str` | required | Target simulation |
| `platform` | `str` | `"parallel"` | `"twitter"`, `"reddit"`, or `"parallel"` |
| `max_rounds` | `int` | `None` | Truncate rounds (appends `--max-rounds` to OASIS command) |
| `enable_graph_memory_update` | `bool` | `False` | Push agent actions to Zep graph in real time |
| `graph_id` | `str` | `None` | Required if graph memory update enabled |

**Process:**
1. Checks no simulation is already running.
2. Reads `simulation_config.json` to determine `total_rounds`.
3. Creates `SimulationRunState(STARTING)` and saves to disk.
4. If graph memory enabled, calls `ZepGraphMemoryManager.create_updater()`.
5. Selects script: `run_twitter_simulation.py`, `run_reddit_simulation.py`, or `run_parallel_simulation.py`.
6. Launches subprocess via `subprocess.Popen` with:
   - `cwd = sim_dir` (so SQLite databases and output files land there)
   - `stdout` / `stderr` → `simulation.log`
   - `PYTHONUTF8=1`, `PYTHONIOENCODING=utf-8` env vars
   - `start_new_session=True` (creates process group for clean termination)
7. Sets state to `RUNNING`, saves.
8. Starts `_monitor_simulation()` in a daemon thread.

---

### `_monitor_simulation(simulation_id)` (private, runs in daemon thread)

Continuously polls two JSONL log files written by OASIS:
- `<sim_dir>/twitter/actions.jsonl`
- `<sim_dir>/reddit/actions.jsonl`

Every 2 seconds while `process.poll() is None`:
1. Calls `_read_action_log()` for each platform.
2. Saves updated state.

After process exits:
- Exit code 0 → `COMPLETED`
- Non-zero → `FAILED` + reads last 2000 chars of `simulation.log` as error
- Calls `ZepGraphMemoryManager.stop_updater()` if graph memory was enabled
- Cleans up process, queue, and file handle references

---

### `_read_action_log(log_path, position, state, platform) → int` (private)

Reads the JSONL file from `position` to end. Returns the new byte position.

**Two types of JSONL entries:**

| Entry has `event_type` key | Behavior |
|-----------------------------|----------|
| `simulation_end` | Marks platform as completed; calls `_check_all_platforms_completed()` |
| `round_end` | Updates per-platform `current_round` and `simulated_hours` |

Normal entries (no `event_type`) are parsed into `AgentAction` objects and passed to `state.add_action()`. If graph memory is enabled, also calls `graph_updater.add_activity_from_dict()`.

---

### `_check_all_platforms_completed(state) → bool` (private)

Detects which platforms are enabled by checking whether `twitter/actions.jsonl` and `reddit/actions.jsonl` exist. Returns `True` only when all enabled platforms have the `twitter_completed` / `reddit_completed` flags set.

---

### `_terminate_process(process, simulation_id, timeout=10)` (private)

Cross-platform process-tree termination:

| OS | Strategy |
|----|----------|
| **Windows** | `taskkill /PID <pid> /T` (terminate tree gracefully), then `/F` if timeout |
| **Unix/Mac** | `os.killpg(pgid, SIGTERM)`, then `SIGKILL` if timeout |

Falls back to `process.terminate()` / `process.kill()` on any failure.

---

### `stop_simulation(simulation_id) → SimulationRunState`

1. Loads state, validates status is `RUNNING` or `PAUSED`.
2. Sets status to `STOPPING`.
3. Calls `_terminate_process()`.
4. Sets status to `STOPPED`, saves, returns updated state.

---

## Files Used at Runtime

```
uploads/simulations/<sim_id>/
    simulation_config.json          ← Read by start_simulation()
    run_state.json                  ← Written every 2 seconds by monitor thread
    simulation.log                  ← stdout+stderr of OASIS subprocess
    twitter/
        actions.jsonl               ← Per-round JSONL written by Twitter OASIS branch
    reddit/
        actions.jsonl               ← Per-round JSONL written by Reddit OASIS branch
```

---

## Related Files

- [`api/simulation.py`](../api/simulation.md) — Calls `start_simulation()`, `stop_simulation()`, `get_run_state()`
- [`services/simulation_manager.py`](simulation_manager.md) — Manages prepare stage before runner is invoked
- [`services/simulation_ipc.py`](simulation_ipc.md) — IPC for interview commands to/from running simulation
- [`services/zep_graph_memory_updater.py`](zep_graph_memory_updater.md) — Optional live graph updates
