# `backend/app/utils/retry.py` — Retry Helpers

## Overview

`retry.py` provides reusable retry logic for unstable external calls such as LLM requests or remote APIs. It supports both decorator-style retries and an object-oriented helper for one-off and batch calls.

The strategy used throughout the file is exponential backoff with optional jitter.

---

## `retry_with_backoff(...)`

A decorator factory for synchronous functions.

Configurable arguments:
- `max_retries`
- `initial_delay`
- `max_delay`
- `backoff_factor`
- `jitter`
- `exceptions`
- `on_retry`

### Returned inner functions

- `decorator(func)` wraps the target function
- `wrapper(*args, **kwargs)` performs the retry loop

Behavior of the wrapper:
1. calls the wrapped function
2. catches only the configured exception types
3. logs each failure
4. computes a backoff delay, optionally randomized
5. runs `on_retry(exception, retry_count)` if provided
6. raises the final exception after the last retry

---

## `retry_with_backoff_async(...)`

Async equivalent of the decorator factory above.

Differences:
- wraps `async def` functions
- uses `await asyncio.sleep(...)` instead of `time.sleep(...)`

The control flow and retry semantics are otherwise the same.

---

## `RetryableAPIClient`

A stateful helper class that stores retry settings once and reuses them for multiple calls.

### `__init__(max_retries=3, initial_delay=1.0, max_delay=30.0, backoff_factor=2.0)`

Stores retry policy parameters.

### `call_with_retry(func, *args, exceptions=(Exception,), **kwargs) -> Any`

Runs one callable with retries.

Behavior:
- repeatedly calls `func(*args, **kwargs)`
- retries on configured exception types
- logs each failed attempt
- raises after the last allowed retry

### `call_batch_with_retry(items, process_func, exceptions=(Exception,), continue_on_failure=True) -> Tuple[list, list]`

Processes many items one by one using `call_with_retry()`.

Returns:
- a list of successful results
- a list of failure records containing index, item, and error string

If `continue_on_failure=False`, the first unrecoverable item aborts the whole batch.

---

## Related Files

- [`utils/llm_client.py`](llm_client.md) is the main type of dependency this retry layer is designed to protect
- [`services/zep_entity_reader.py`](../services/zep_entity_reader.md) and other service modules use retry patterns for remote calls
