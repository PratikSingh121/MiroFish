# `backend/app/api/graph.py` — Graph API Routes

## Overview

`graph.py` defines all HTTP endpoints under the `/api/graph` prefix (mounted via `graph_bp`). It orchestrates the **first two steps** of the MiroFish workflow:

1. **Step 1a — Ontology Generation** (`POST /api/graph/ontology/generate`): Receives document uploads and a simulation requirement, calls the `OntologyGenerator` to analyse them with an LLM, and saves the result as part of a new `Project`.
2. **Step 1b — Graph Build** (`POST /api/graph/build`): Reads the project's extracted text and ontology, then starts an asynchronous background task that feeds the data to Zep via `GraphBuilderService`.
3. **Project / Task management** endpoints: CRUD for `Project` objects, polling endpoints for long-running `Task` objects, and graph data endpoints.

All long-running operations are performed in **background threads** so that the HTTP response is returned immediately with a `task_id` that the frontend can poll.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `OntologyGenerator` | [`services/ontology_generator.py`](../services/ontology_generator.md) | LLM-based ontology extraction |
| `GraphBuilderService` | [`services/graph_builder.py`](../services/graph_builder.md) | Zep graph construction |
| `TextProcessor` | [`services/text_processor.py`](../services/text_processor.md) | Text chunking & preprocessing |
| `FileParser` | [`utils/file_parser.py`](../utils/file_parser.md) | PDF/MD/TXT extraction |
| `TaskManager`, `TaskStatus` | [`models/task.py`](../models/task.md) | In-memory async task tracking |
| `ProjectManager`, `ProjectStatus` | [`models/project.py`](../models/project.md) | Persistent project state |
| `Config` | [`config.py`](../config.md) | File upload settings |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Helper Function

### `allowed_file(filename)`

```python
def allowed_file(filename: str) -> bool:
```

Validates that an uploaded file has an extension listed in `Config.ALLOWED_EXTENSIONS` (`pdf`, `md`, `txt`, `markdown`). Used to filter files before processing. Handles `None` and files without an extension safely.

---

## Endpoint Reference

### Project Management

#### `GET /api/graph/project/<project_id>`
**`get_project(project_id)`**

Returns the full project dictionary for the given `project_id`. Returns HTTP 404 if the project does not exist.

---

#### `GET /api/graph/project/list`
**`list_projects()`**

Returns a list of all projects ordered by creation time (newest first). Optional `limit` query parameter (default 50).

---

#### `DELETE /api/graph/project/<project_id>`
**`delete_project(project_id)`**

Permanently deletes a project and all its files from disk via `ProjectManager.delete_project()`.

---

#### `POST /api/graph/project/<project_id>/reset`
**`reset_project(project_id)`**

Resets a project back to the `ONTOLOGY_GENERATED` (or `CREATED`) state so the graph can be rebuilt without losing the ontology. Clears `graph_id`, `graph_build_task_id`, and `error`.

---

### Step 1a — Ontology Generation

#### `POST /api/graph/ontology/generate`
**`generate_ontology()`**

**Request:** `multipart/form-data`

| Field | Required | Description |
|-------|----------|-------------|
| `files` | Yes | One or more PDF/MD/TXT files |
| `simulation_requirement` | Yes | Natural-language description of what to simulate/predict |
| `project_name` | No | Display name for the project |
| `additional_context` | No | Extra background information for the LLM |

**What it does:**
1. Creates a new `Project` via `ProjectManager.create_project()`.
2. Saves each uploaded file to a per-project directory.
3. Extracts text from each file using `FileParser.extract_text()`.
4. Preprocesses text with `TextProcessor.preprocess_text()`.
5. Calls `OntologyGenerator.generate()` to produce `entity_types` and `edge_types`.
6. Saves the ontology back to the project and sets status to `ONTOLOGY_GENERATED`.

**Response:**
```json
{
  "success": true,
  "data": {
    "project_id": "proj_xxxx",
    "ontology": { "entity_types": [...], "edge_types": [...] },
    "analysis_summary": "...",
    "files": [...],
    "total_text_length": 12345
  }
}
```

---

### Step 1b — Graph Build

#### `POST /api/graph/build`
**`build_graph()`**

**Request:** JSON body

| Field | Required | Description |
|-------|----------|-------------|
| `project_id` | Yes | ID from Step 1a |
| `graph_name` | No | Display name for the Zep graph |
| `chunk_size` | No | Characters per chunk (default from `Config.DEFAULT_CHUNK_SIZE`) |
| `chunk_overlap` | No | Overlap between chunks (default from `Config.DEFAULT_CHUNK_OVERLAP`) |
| `force` | No | Force rebuild even if already building/completed |

**What it does:**
1. Validates project exists and has an ontology.
2. Prevents duplicate kicks (if `status == GRAPH_BUILDING` and `force` is not set).
3. Creates a `Task` via `TaskManager.create_task()`.
4. Launches a background thread that:
   - Creates a `GraphBuilderService`.
   - Splits text into chunks with `TextProcessor.split_text()`.
   - Calls `GraphBuilderService.build_graph_async()` (or rather the sync steps inline).
   - Reports progress via `task_manager.update_task()`.
   - On success sets project status to `GRAPH_COMPLETED` and saves the `graph_id`.
   - On failure sets project status to `FAILED`.

**Response:** Returns immediately with `task_id`.

---

### Task & Graph Status

#### `GET /api/graph/task/<task_id>`
**`get_task_status(task_id)`**

Returns the current state of a background task (progress 0–100, status, message, result/error).

---

#### `GET /api/graph/data/<graph_id>`
**`get_graph_data(graph_id)`**

Fetches all nodes and edges from the Zep graph and returns them formatted for the frontend's D3.js visualization in `GraphPanel`.

---

## Error Handling

All endpoints wrap their logic in `try/except Exception` and return:
```json
{
  "success": false,
  "error": "...",
  "traceback": "..."
}
```
with an appropriate HTTP status code (400/404/500).

---

## Related Files

- [`services/ontology_generator.py`](../services/ontology_generator.md) — generates entity/edge type ontology
- [`services/graph_builder.py`](../services/graph_builder.md) — builds the Zep knowledge graph
- [`services/text_processor.py`](../services/text_processor.md) — text chunking
- [`utils/file_parser.py`](../utils/file_parser.md) — document text extraction
- [`models/project.py`](../models/project.md) — `Project` and `ProjectManager`
- [`models/task.py`](../models/task.md) — `Task` and `TaskManager`
- Frontend: [`frontend/src/api/graph.js`](../../../frontend/src/api/graph.md)
