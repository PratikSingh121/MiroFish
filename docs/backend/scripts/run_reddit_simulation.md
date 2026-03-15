# `backend/scripts/run_reddit_simulation.py` — Reddit OASIS Runner

## Overview

This script runs a single-platform Reddit simulation from a generated config file. It is the Reddit-only counterpart to the dual-platform runner.

Key responsibilities:
- load environment variables and OASIS dependencies
- configure script-specific logging
- run the Reddit simulation according to the generated parameters
- expose IPC-based interviews and shutdown while the environment remains alive

---

## Logging Helpers

### `UnicodeFormatter`
Converts escaped Unicode sequences into readable characters in log output.

### `MaxTokensWarningFilter`
Suppresses noisy camel-ai warnings about unset `max_tokens`.

### `setup_oasis_logging(log_dir)`
Creates dedicated OASIS log files for internal subsystems.

---

## IPC Support

### `CommandType`
Defines `INTERVIEW`, `BATCH_INTERVIEW`, and `CLOSE_ENV` constants.

### `IPCHandler`
Owns the file-based command/response loop for a single Reddit environment.

Responsibilities:
- update `env_status.json`
- poll `ipc_commands/`
- write `ipc_responses/`
- run single-agent and batch interviews
- close the environment on command

---

## `RedditSimulationRunner`

The main orchestration class for this script.

At a high level it:
- loads config and agent profiles
- builds the Reddit OASIS environment
- chooses active agents each round
- executes their actions
- logs actions for backend monitoring
- exposes the finished environment for post-run interviews

---

## Top-Level Functions

### `main()`
Parses CLI arguments, loads config, constructs `RedditSimulationRunner`, runs the simulation, and optionally leaves the environment alive for later IPC commands.

### `setup_signal_handlers()`
Registers graceful shutdown behavior.

---

## Related Files

- [`action_logger.py`](action_logger.md)
- [`run_parallel_simulation.py`](run_parallel_simulation.md)
- [`../app/services/simulation_runner.py`](../app/services/simulation_runner.md)
- [`../app/services/simulation_ipc.py`](../app/services/simulation_ipc.md)
