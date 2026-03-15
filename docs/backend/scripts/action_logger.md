# `backend/scripts/action_logger.py` — Simulation Action Logging

## Overview

`action_logger.py` provides the logging primitives used by the OASIS runner scripts. It writes action logs in the JSONL format expected by the backend runtime monitor and also manages the main `simulation.log` file.

This script is part of the bridge between standalone OASIS execution and Flask-side status polling.

---

## `PlatformActionLogger`

Handles one platform at a time (`twitter` or `reddit`).

### `__init__(platform, base_dir)`
Sets log directory and file path for `<base_dir>/<platform>/actions.jsonl`.

### `_ensure_dir()`
Creates the platform subdirectory.

### `log_action(...)`
Appends one normal agent action record.

### `log_round_start(round_num, simulated_hour)`
Writes a control record with `event_type = 'round_start'`.

### `log_round_end(round_num, actions_count)`
Writes a control record with `event_type = 'round_end'`.

### `log_simulation_start(config)`
Writes the opening event for a platform run.

### `log_simulation_end(total_rounds, total_actions)`
Writes the closing event used by the backend to detect completion.

---

## `SimulationLogManager`

Higher-level log manager for one simulation directory.

### `__init__(simulation_dir)`
Prepares lazy platform loggers and configures the shared `simulation.log` logger.

### `_setup_main_logger()`
Creates the main Python logger, attaches file and console handlers, and disables propagation.

### `get_twitter_logger()` / `get_reddit_logger()`
Lazy-creates a `PlatformActionLogger` for each platform.

### `log(message, level='info')`
Routes generic messages to the main logger.

### Convenience wrappers
- `info()`
- `warning()`
- `error()`
- `debug()`

---

## `ActionLogger` (Legacy Compatibility)

Older single-file logger that stores `platform` directly in each entry instead of separating by subdirectory.

Methods:
- `__init__(log_path)`
- `_ensure_dir()`
- `log_action(...)`
- `log_round_start(round_num, simulated_hour, platform)`
- `log_round_end(round_num, actions_count, platform)`

### `get_logger(log_path=None) -> ActionLogger`

Legacy factory returning an `ActionLogger` instance.

---

## Related Files

- [`run_parallel_simulation.py`](run_parallel_simulation.md)
- [`run_reddit_simulation.py`](run_reddit_simulation.md)
- [`run_twitter_simulation.py`](run_twitter_simulation.md)
- [`../app/services/simulation_runner.py`](../app/services/simulation_runner.md)
