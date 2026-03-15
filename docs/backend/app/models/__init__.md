# `backend/app/models/__init__.py` — Model Package Exports

## Overview

This file is the public export surface for the backend model package. It does not define new logic; instead, it re-exports the project and task model types so other modules can import them from `app.models` instead of reaching into individual files.

---

## Imports and Re-exports

From [`task.py`](task.md):
- `TaskManager`
- `TaskStatus`

From [`project.py`](project.md):
- `Project`
- `ProjectStatus`
- `ProjectManager`

### `__all__`

The `__all__` list defines the official package-level exports for wildcard imports and communicates the intended public API of the model package.

---

## Related Files

- [`project.py`](project.md)
- [`task.py`](task.md)
