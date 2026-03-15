# `backend/app/utils/logger.py` — Logger Setup

## Overview

`logger.py` centralizes logging for the backend. It creates named loggers that write both to rotating daily log files and to the console.

Two design goals are visible in this file:
- provide rich file logs for debugging
- avoid Windows console encoding issues when Chinese text is printed

---

## Constants

### `LOG_DIR`

Points to `backend/logs/` relative to the app package root.

---

## Functions

### `_ensure_utf8_stdout()`

Windows-only helper that reconfigures `sys.stdout` and `sys.stderr` to UTF-8 when possible.

This prevents mojibake in console output.

### `setup_logger(name='mirofish', level=logging.DEBUG) -> logging.Logger`

Creates and configures a logger.

Behavior:
1. creates `LOG_DIR` if needed
2. gets a named logger
3. disables propagation to the root logger to avoid duplicates
4. returns early if handlers already exist
5. attaches a `RotatingFileHandler` with detailed formatting
6. attaches a console handler with simpler formatting and INFO level

File logs are rotated at 10 MB with 5 backups.

### `get_logger(name='mirofish') -> logging.Logger`

Convenience getter that calls `setup_logger()` only if the named logger has not already been configured.

### Module-level convenience functions

The module also exposes direct pass-through helpers bound to the default logger:
- `debug()`
- `info()`
- `warning()`
- `error()`
- `critical()`

These are shortcuts for places that do not need a custom named logger.

---

## Related Files

- Every backend module imports `get_logger()` from here
- [`utils/retry.py`](retry.md) uses it for retry logs
- [`services/report_agent.py`](../services/report_agent.md) adds an extra file handler for per-report console logs
