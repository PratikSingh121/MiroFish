# `backend/app/models/task.py` — Async Task Tracking

## Overview

`task.py` provides a **thread-safe, in-memory task registry** for tracking long-running background operations (graph build, simulation prepare, report generate). Because these operations can take minutes, the API pattern is:

1. Client sends a request → server immediately returns a `task_id`.
2. Client polls `GET /api/graph/task/<task_id>` periodically.
3. Background thread updates the task via `TaskManager.update_task()`.

> **Important:** `TaskManager` is a **singleton** stored purely in memory. Tasks are lost on server restart. This is intentional for the current architecture — the actual results are persisted to disk by the services themselves.

---

## `TaskStatus` Enum

```python
class TaskStatus(str, Enum):
    PENDING    = "pending"     # Created, not yet started
    PROCESSING = "processing"  # Background thread active
    COMPLETED  = "completed"   # Finished successfully
    FAILED     = "failed"      # Finished with an error
```

---

## `Task` Dataclass

```python
@dataclass
class Task:
    task_id: str
    task_type: str                    # e.g. "graph_build", "simulation_prepare"
    status: TaskStatus
    created_at: datetime
    updated_at: datetime
    progress: int = 0                 # 0-100
    message: str = ""                 # Human-readable status message
    result: Optional[Dict] = None     # Final result payload
    error: Optional[str] = None       # Error message if status==FAILED
    metadata: Dict = {}               # Arbitrary key-value context
    progress_detail: Dict = {}        # Detailed sub-progress breakdowns
```

### `Task.to_dict()`

Serializes the task to a dict suitable for JSON API responses. Timestamps are converted to ISO 8601 strings.

---

## `TaskManager` Class

### Singleton Pattern

```python
class TaskManager:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                ...
```

`TaskManager()` always returns the **same instance** regardless of where it is called. This means all parts of the application share one task registry.

The internal task dictionary `_tasks: Dict[str, Task]` and a `_task_lock: threading.Lock()` are stored on the singleton instance, providing thread safety for concurrent reads and writes from multiple background threads.

---

### Method Reference

#### `create_task(task_type, metadata=None) → str`

Creates a new `Task` with status `PENDING`, stores it in `_tasks`, and returns the UUID `task_id`.

```python
task_id = task_manager.create_task(
    task_type="graph_build",
    metadata={"graph_name": "My Graph", "text_length": 45000}
)
```

---

#### `get_task(task_id) → Optional[Task]`

Thread-safe lookup. Returns `None` if the task ID is not found.

---

#### `update_task(task_id, status=None, progress=None, message=None, result=None, error=None, progress_detail=None) → None`

Thread-safe update. Any parameter that is `None` is left unchanged. Always updates `updated_at`. This is the method called frequently by background threads to report progress:

```python
task_manager.update_task(
    task_id,
    progress=45,
    message="Processing chunk 45/100"
)
```

---

#### `complete_task(task_id, result) → None`

Convenience wrapper around `update_task` that sets:
- `status = COMPLETED`
- `progress = 100`
- `message = "任务完成"`
- `result = result`

---

#### `fail_task(task_id, error) → None`

Convenience wrapper that sets:
- `status = FAILED`
- `message = "任务失败"`
- `error = error`

---

#### `list_tasks(task_type=None) → list`

Returns all tasks (or tasks of a specific type) sorted newest-first as dicts.

---

#### `cleanup_old_tasks(max_age_hours=24) → None`

Removes completed/failed tasks older than `max_age_hours` from memory. Should be called periodically to prevent unbounded memory growth.

---

## Thread Safety

All methods acquire `_task_lock` (a `threading.Lock`) before reading or writing `_tasks`. This prevents race conditions when multiple background threads update the same registry simultaneously.

---

## Related Files

- [`api/graph.py`](../api/graph.md) — creates tasks for graph build operations
- [`api/simulation.py`](../api/simulation.md) — creates tasks for simulation prepare
- [`api/report.py`](../api/report.md) — creates tasks for report generation
- [`services/graph_builder.py`](../services/graph_builder.md) — calls `update_task` via the task manager stored on the builder instance
