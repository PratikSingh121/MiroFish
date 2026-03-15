# `backend/app/models/project.py` — Project Data Model & Manager

## Overview

`project.py` defines the **persistent project state** used throughout the MiroFish workflow. A `Project` represents a single "run" of the engine: it tracks uploaded files, the LLM-generated ontology, the Zep graph ID, and the overall workflow status.

Projects are stored on disk as JSON files inside `backend/uploads/projects/<project_id>/project.json`. This makes them durable across server restarts (unlike `TaskManager` which is in-memory only).

---

## `ProjectStatus` Enum

```python
class ProjectStatus(str, Enum):
    CREATED            # Files uploaded, no ontology yet
    ONTOLOGY_GENERATED # LLM ontology produced
    GRAPH_BUILDING     # Zep graph build in progress
    GRAPH_COMPLETED    # Graph ready — simulation can proceed
    FAILED             # An error occurred
```

The status flows: `CREATED → ONTOLOGY_GENERATED → GRAPH_BUILDING → GRAPH_COMPLETED`.

---

## `Project` Dataclass

A `@dataclass` representing a single project.

```python
@dataclass
class Project:
    project_id: str        # e.g. "proj_abc123456789"
    name: str
    status: ProjectStatus
    created_at: str        # ISO 8601 timestamp
    updated_at: str
    
    files: List[Dict]      # [{filename, size}, ...]
    total_text_length: int
    
    ontology: Optional[Dict]          # {entity_types, edge_types}
    analysis_summary: Optional[str]
    
    graph_id: Optional[str]           # Zep graph ID e.g. "mirofish_..."
    graph_build_task_id: Optional[str]
    
    simulation_requirement: Optional[str]
    chunk_size: int         # default 500
    chunk_overlap: int      # default 50
    
    error: Optional[str]
```

### `Project.to_dict()`

Serializes the project to a plain Python dict for JSON storage. Converts `ProjectStatus` enum values to their string representation.

### `Project.from_dict(data)`

Class method. Deserializes a project from a dict (as loaded from `project.json`). Handles backward compatibility by defaulting missing fields.

---

## `ProjectManager` Class

A **class-only manager** (all methods are `@classmethod`). No instances are created. Uses the filesystem for storage.

### Storage Layout

```
backend/uploads/projects/
└── proj_abc123456789/
    ├── project.json          ← Project metadata (ProjectManager.save_project)
    ├── extracted_text.txt    ← Concatenated text from all uploaded files
    └── files/
        ├── a1b2c3d4.pdf      ← Uploaded files with UUID-prefixed names
        └── ...
```

### Class Attribute

```python
PROJECTS_DIR = os.path.join(Config.UPLOAD_FOLDER, 'projects')
```

### Private Helpers

| Method | Returns | Description |
|--------|---------|-------------|
| `_ensure_projects_dir()` | None | Creates `PROJECTS_DIR` if it doesn't exist |
| `_get_project_dir(project_id)` | str | `PROJECTS_DIR/<project_id>/` |
| `_get_project_meta_path(project_id)` | str | `<project_dir>/project.json` |
| `_get_project_files_dir(project_id)` | str | `<project_dir>/files/` |
| `_get_project_text_path(project_id)` | str | `<project_dir>/extracted_text.txt` |

### Public Methods

#### `create_project(name="Unnamed Project") → Project`

Creates a new project with a UUID-based ID (`proj_<12 hex chars>`), creates the directory structure on disk, saves the initial `project.json`, and returns the `Project` object.

---

#### `save_project(project) → None`

Serializes the project via `project.to_dict()` and writes to `project.json`. Also updates `project.updated_at` to the current time.

Called by:
- [`api/graph.py`](../api/graph.md) after ontology generation, graph build updates
- [`api/simulation.py`](../api/simulation.md) (indirectly via ProjectManager)

---

#### `get_project(project_id) → Optional[Project]`

Reads `project.json` from disk and deserializes via `Project.from_dict()`. Returns `None` if the file doesn't exist.

---

#### `list_projects(limit=50) → List[Project]`

Lists all project directories, loads each project, and returns them sorted by `created_at` descending (newest first). Silently skips any corrupted project directories.

---

#### `delete_project(project_id) → bool`

Uses `shutil.rmtree()` to delete the entire project directory (including all uploaded files and extracted text). Returns `False` if the directory doesn't exist.

---

#### `save_file_to_project(project_id, file_storage, original_filename) → Dict`

Saves a Flask `FileStorage` object to the project's `files/` directory. Generates a short UUID-based filename to avoid conflicts. Returns a dict with `original_filename`, `saved_filename`, `path`, and `size`.

---

#### `save_extracted_text(project_id, text) → None`

Writes the extracted document text to `extracted_text.txt`. Called after all uploaded files are parsed.

---

#### `get_extracted_text(project_id) → Optional[str]`

Reads and returns the contents of `extracted_text.txt`. Returns `None` if the file doesn't exist. Used by `api/graph.py` when building the graph.

---

#### `get_project_files(project_id) → List[str]`

Returns a list of full paths to all files in the project's `files/` directory.

---

## Related Files

- [`api/graph.py`](../api/graph.md) — creates and updates projects during graph workflow
- [`api/simulation.py`](../api/simulation.md) — reads project data for simulation setup
- [`api/report.py`](../api/report.md) — reads `simulation_requirement` and `graph_id` for report generation
- [`config.py`](../config.md) — provides `UPLOAD_FOLDER` base path
