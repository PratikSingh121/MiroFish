# `backend/app/services/zep_graph_memory_updater.py` — Zep Graph Memory Updater

## Overview

`zep_graph_memory_updater.py` streams agent behavior from a running simulation back into the Zep graph as new text episodes. Its job is to translate low-level OASIS action logs into natural-language memory statements that Zep can parse into entities and relationships.

The design has two layers:
- `AgentActivity` converts one action into human-readable text
- `ZepGraphMemoryUpdater` batches and sends those descriptions to Zep in a background worker thread
- `ZepGraphMemoryManager` keeps one updater per simulation

---

## `AgentActivity`

Fields:
- `platform`
- `agent_id`
- `agent_name`
- `action_type`
- `action_args`
- `round_num`
- `timestamp`

### `to_episode_text() -> str`

Dispatches to an action-specific formatter and returns text in this shape:

```text
<agent_name>: <natural language activity description>
```

The goal is to make Zep infer graph updates from ordinary language instead of simulation-specific JSON.

### Action-specific formatters

Each private helper translates one action family into fluent Chinese text with as much context as possible from `action_args`.

- `_describe_create_post()`
- `_describe_like_post()`
- `_describe_dislike_post()`
- `_describe_repost()`
- `_describe_quote_post()`
- `_describe_follow()`
- `_describe_create_comment()`
- `_describe_like_comment()`
- `_describe_dislike_comment()`
- `_describe_search()`
- `_describe_search_user()`
- `_describe_mute()`
- `_describe_generic()` fallback for unknown action types

These helpers intentionally include referenced post content, author names, comment text, and usernames whenever available, because richer text gives Zep better extraction input.

---

## `ZepGraphMemoryUpdater`

### Key class settings

| Setting | Meaning |
|---------|---------|
| `BATCH_SIZE = 5` | send activities to Zep in groups of five per platform |
| `PLATFORM_DISPLAY_NAMES` | maps `twitter`/`reddit` to UI-facing names |
| `SEND_INTERVAL = 0.5` | delay between batch sends |
| `MAX_RETRIES = 3` | retry count for Zep writes |
| `RETRY_DELAY = 2` | base retry delay |

### `__init__(graph_id, api_key=None)`

Initializes the Zep client, queue, per-platform buffers, worker-thread state, and cumulative counters.

Raises `ValueError` if `ZEP_API_KEY` is missing.

### `_get_platform_display_name(platform) -> str`

Returns a readable platform label for logging.

### `start()`

Starts the background worker thread if it is not already running.

### `stop()`

Stops the worker, flushes remaining queued activities, waits for the worker thread to exit, and logs final stats.

### `add_activity(activity)`

Enqueues one `AgentActivity`.

Special rule: `DO_NOTHING` events are skipped and counted in `skipped_count` because they do not represent meaningful memory updates.

### `add_activity_from_dict(data, platform)`

Builds an `AgentActivity` from an action-log dict and forwards it to `add_activity()`.

Ignores entries that contain `event_type`, because those are control events such as `round_end`, not agent behavior.

### `_worker_loop()`

Background thread loop.

Behavior:
1. drains the activity queue
2. groups items into platform-specific buffers
3. sends a batch once one platform buffer reaches `BATCH_SIZE`
4. continues until both `_running` is false and the queue is empty

### `_send_batch_activities(activities, platform)`

Converts a batch into one newline-joined text block and calls `self.client.graph.add(...)`.

Retries on failure and updates counters:
- `total_sent`
- `total_items_sent`
- `failed_count`

### `_flush_remaining()`

Moves all still-queued items into buffers and sends every remaining platform buffer, even if it has fewer than `BATCH_SIZE` items.

### `get_stats() -> Dict[str, Any]`

Returns runtime metrics including queue size, per-platform buffer sizes, sent counts, and failure counts.

---

## `ZepGraphMemoryManager`

Singleton-like manager for all updater instances.

### `create_updater(simulation_id, graph_id) -> ZepGraphMemoryUpdater`

Stops any existing updater for that simulation, creates a fresh one, starts it, stores it, and returns it.

### `get_updater(simulation_id) -> Optional[ZepGraphMemoryUpdater]`

Fetches the updater currently associated with the simulation.

### `stop_updater(simulation_id)`

Stops and removes one updater.

### `stop_all()`

Stops every registered updater exactly once using the `_stop_all_done` guard.

### `get_all_stats() -> Dict[str, Dict[str, Any]]`

Returns `get_stats()` for every active updater, keyed by simulation ID.

---

## Related Files

- [`services/simulation_runner.py`](simulation_runner.md) feeds action logs into this updater during live runs
- [`services/simulation_manager.py`](simulation_manager.md) prepares the simulation before live updates begin
- [`services/zep_tools.py`](zep_tools.md) later reads the enriched graph during report generation
