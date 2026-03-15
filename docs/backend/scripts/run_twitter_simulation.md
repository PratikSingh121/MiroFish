# `backend/scripts/run_twitter_simulation.py` — Twitter OASIS Runner

## Overview

This script runs a single-platform Twitter simulation from a generated config file. It mirrors the Reddit runner but targets the Twitter action set and Twitter-specific OASIS environment.

It is used when the backend wants only the Twitter branch instead of a dual-platform run.

---

## Logging Helpers

### `UnicodeFormatter`
Makes Unicode-heavy logs readable.

### `MaxTokensWarningFilter`
Filters camel-ai `max_tokens` warnings.

### `setup_oasis_logging(log_dir)`
Configures file-based subsystem logging for OASIS internals.

---

## IPC Support

### `CommandType`
Constant holder for interview and environment-control commands.

### `IPCHandler`
Single-platform IPC manager for the Twitter environment.

Core responsibilities:
- publish environment status
- poll command files
- execute single-agent and batch interviews
- write response files
- process graceful close requests

---

## `TwitterSimulationRunner`

The main orchestration class.

Responsibilities:
- load config and Twitter profile CSV data
- initialize the Twitter OASIS environment
- select active agents for each round
- run agent actions
- persist action records through the action logger
- keep the environment alive for IPC if wait mode is enabled

---

## Top-Level Functions

### `main()`
CLI entry point that loads config, creates the runner, executes the simulation, and optionally waits for later commands.

### `setup_signal_handlers()`
Registers graceful signal-based shutdown.

---

## Related Files

- [`action_logger.py`](action_logger.md)
- [`run_parallel_simulation.py`](run_parallel_simulation.md)
- [`../app/services/simulation_runner.py`](../app/services/simulation_runner.md)
- [`../app/services/simulation_ipc.py`](../app/services/simulation_ipc.md)
