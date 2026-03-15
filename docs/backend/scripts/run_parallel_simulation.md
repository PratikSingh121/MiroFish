# `backend/scripts/run_parallel_simulation.py` — Dual-Platform OASIS Runner

## Overview

This script runs Twitter and Reddit simulations in parallel from the same `simulation_config.json`. It is the executable backend target used for live dual-platform simulations.

It is responsible for:
- environment and UTF-8 setup, especially on Windows
- OASIS initialization
- per-platform action logging
- live IPC handling for interviews and environment shutdown
- round scheduling based on generated config
- keeping the environment alive after simulation completion unless `--no-wait` is used

---

## Startup and Environment Helpers

### `MaxTokensWarningFilter`
Filters noisy camel-ai warnings about `max_tokens`.

### `disable_oasis_logging()`
Suppresses verbose internal OASIS logs so the script’s own structured logging remains readable.

### `init_logging_for_simulation(simulation_dir)`
Cleans old log folders and prepares simulation logging.

### `CommandType`
Lightweight constant holder for IPC command names.

---

## `ParallelIPCHandler`

Manages interview and close-environment commands while both platform environments are alive.

Important responsibilities:
- monitor `ipc_commands/`
- write matching files into `ipc_responses/`
- route interviews to Twitter, Reddit, or both
- expose environment status through `env_status.json`

Internally it contains helper methods such as platform lookup and single-platform interview execution.

---

## Configuration and Data Helpers

### `load_config(config_path)`
Reads the generated simulation config JSON.

### `get_agent_names_from_config(config)`
Builds an ID-to-name mapping for display and action logs.

### `fetch_new_actions_from_db(...)`
Reads newly generated actions from the simulation database.

### `_enrich_action_context(...)`
Adds post, user, and comment context to action records before they are logged.

### `_get_post_info(...)`
Looks up referenced post details.

### `_get_user_name(...)`
Looks up a user name for action enrichment.

### `_get_comment_info(...)`
Looks up comment details for comment-related actions.

### `create_model(config, use_boost=False)`
Creates the LLM model object used by OASIS agents.

### `get_active_agents_for_round(...)`
Determines which agents should be active for a given round based on the generated simulation parameters.

---

## `PlatformSimulation`

Small holder for per-platform runtime state used by the parallel runner.

---

## Main Execution Coroutines

### `run_twitter_simulation(...)`
Runs one Twitter branch inside the parallel script.

### `run_reddit_simulation(...)`
Runs one Reddit branch inside the parallel script.

Both functions follow the same pattern:
- initialize platform-specific OASIS env and graph
- schedule agents by round
- execute actions
- persist actions via the platform logger
- keep enough state for the IPC handler and backend monitor

### `main()`
Top-level coroutine.

Responsibilities:
- parse CLI arguments
- load config
- initialize logging
- choose Twitter-only, Reddit-only, or full parallel execution
- run both branches concurrently
- optionally enter post-run wait mode so interviews remain available

### `setup_signal_handlers(loop=None)`
Registers graceful shutdown handlers.

---

## Why This Script Matters

This script is the actual engine behind the live simulation experience. The backend launches it as a subprocess, then monitors its log files and sends IPC commands to it.

---

## Related Files

- [`action_logger.py`](action_logger.md)
- [`../app/services/simulation_runner.py`](../app/services/simulation_runner.md)
- [`../app/services/simulation_ipc.py`](../app/services/simulation_ipc.md)
