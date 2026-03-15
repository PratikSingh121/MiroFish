# `backend/app/api/report.py` — Report API Routes

## Overview

`report.py` defines all HTTP endpoints under the `/api/report` prefix. It handles the **entire report lifecycle**:

1. **Generating a report** (async) — starts the `ReportAgent` in a background thread.
2. **Polling generation progress** — status endpoint to monitor the async task.
3. **Fetching the completed report** — by `report_id` or by `simulation_id`.
4. **Downloading the report** — as a Markdown file.
5. **Listing reports** — with optional filtering.
6. **Chatting with the Report Agent** — allows conversational follow-up questions after the report is generated.
7. **Accessing detailed logs** — agent decision log (JSONL) and console log.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `ReportAgent`, `ReportManager`, `ReportStatus` | [`services/report_agent.py`](../services/report_agent.md) | Report generation and storage |
| `SimulationManager` | [`services/simulation_manager.py`](../services/simulation_manager.md) | Look up simulation state |
| `ProjectManager` | [`models/project.py`](../models/project.md) | Retrieve project configuration |
| `TaskManager`, `TaskStatus` | [`models/task.py`](../models/task.md) | Track async report generation tasks |
| `Config` | [`config.py`](../config.md) | Report storage path configuration |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Endpoint Reference

### Report Generation

#### `POST /api/report/generate`
**`generate_report()`**

**What it does:**
1. Accepts `simulation_id` (required) and `force_regenerate` (optional, default `false`).
2. Looks up the simulation via `SimulationManager.get_simulation()`.
3. If a completed report already exists and `force_regenerate=false`, returns the existing `report_id` immediately.
4. Retrieves the project via `ProjectManager.get_project()` to get `graph_id` and `simulation_requirement`.
5. Pre-generates a `report_id` so the frontend can reference the report before it's complete.
6. Creates an async task via `TaskManager.create_task()`.
7. Launches a daemon thread that:
   - Creates a `ReportAgent(graph_id, simulation_id, simulation_requirement)`.
   - Calls `agent.generate_report(progress_callback=..., report_id=report_id)`.
   - Saves the result via `ReportManager.save_report()`.
   - Marks the task as completed or failed.

**Response:** Returns immediately with `task_id`, `report_id`, and `status="generating"`.

---

#### `POST /api/report/generate/status`
**`get_generate_status()`**

Polls the progress of a running report generation task.

Accepts either:
- `task_id` — the task ID returned from `generate_report`
- `simulation_id` — checks directly in `ReportManager` for a completed report

If the report is already completed (`ReportStatus.COMPLETED`), returns `progress=100` without needing a `task_id`.

---

### Report Retrieval

#### `GET /api/report/<report_id>`
**`get_report(report_id)`**

Returns the full report object via `ReportManager.get_report(report_id)`. The report includes the Markdown content, the structured outline, generation timestamps, and status.

---

#### `GET /api/report/by-simulation/<simulation_id>`
**`get_report_by_simulation(simulation_id)`**

Looks up the report associated with a simulation. Returns HTTP 404 with `"has_report": false` if no report exists yet.

---

#### `GET /api/report/list`
**`list_reports()`**

Returns a paginated list of reports. Optional query params:
- `simulation_id` — filter by simulation
- `limit` — max results (default 50)

---

#### `GET /api/report/<report_id>/download`
**`download_report(report_id)`**

Writes the report's Markdown content to a temporary file and returns it as a file download with `Content-Disposition: attachment`.

---

### Report Chat

#### `POST /api/report/chat`
**`chat_with_report()`**

Allows users to have a conversational follow-up with the Report Agent after the report has been generated.

**Request:**
```json
{
  "simulation_id": "sim_xxxx",
  "message": "What was the most surprising finding?",
  "chat_history": [...]
}
```

The `chat_history` is a list of `{"role": "user"|"assistant", "content": "..."}` objects for multi-turn context.

Internally calls `ReportAgent.chat()` which uses the same Zep tool set as report generation but in a conversational mode.

---

### Log Endpoints

#### `GET /api/report/<report_id>/agent-log`
**`get_agent_log(report_id)`**

Returns entries from the `agent_log.jsonl` file starting from line `from_line` (for incremental loading). The agent log records every thought, tool call, and tool result from the ReACT loop.

#### `GET /api/report/<report_id>/console-log`
**`get_console_log(report_id)`**

Similar to agent-log but returns the plain-text console output captured during report generation.

---

## Error Handling

All endpoints catch `Exception` and return:
```json
{
  "success": false,
  "error": "...",
  "traceback": "..."
}
```

---

## Related Files

- [`services/report_agent.py`](../services/report_agent.md) — `ReportAgent`, `ReportManager`, `ReportStatus`
- [`services/simulation_manager.py`](../services/simulation_manager.md) — look up simulation
- [`models/project.py`](../models/project.md) — get graph ID and simulation requirement
- [`models/task.py`](../models/task.md) — async task management
- Frontend: [`frontend/src/api/report.js`](../../../frontend/src/api/report.md)
