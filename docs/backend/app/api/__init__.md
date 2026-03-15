# `backend/app/api/__init__.py` — API Blueprints Registry

## Overview

This file is the **entry point for the `api` package**. It creates the three Flask `Blueprint` objects that group all API routes and then imports the route modules so that their `@bp.route(...)` decorators are registered.

Flask Blueprints work like mini-applications: they collect routes, error handlers, and other app functionality, and are later mounted onto the main `Flask` app object with a URL prefix.

---

## Blueprints Defined

| Blueprint Variable | Blueprint Name | Mounted At (set in `app/__init__.py`) |
|--------------------|---------------|--------------------------------------|
| `graph_bp` | `'graph'` | `/api/graph` |
| `simulation_bp` | `'simulation'` | `/api/simulation` |
| `report_bp` | `'report'` | `/api/report` |

---

## Import Side-Effects

```python
from . import graph       # registers all @graph_bp.route(...)
from . import simulation  # registers all @simulation_bp.route(...)
from . import report      # registers all @report_bp.route(...)
```

These imports are placed **after** the Blueprint objects are created (a common Flask pattern) to avoid circular import issues. The `# noqa` comments silence linter warnings about imports-not-at-top.

---

## Related Files

- [`backend/app/__init__.py`](../__init__.md) — calls `app.register_blueprint()` for each blueprint
- [`backend/app/api/graph.py`](graph.md) — graph-related routes on `graph_bp`
- [`backend/app/api/simulation.py`](simulation.md) — simulation-related routes on `simulation_bp`
- [`backend/app/api/report.py`](report.md) — report-related routes on `report_bp`
