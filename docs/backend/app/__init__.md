# `backend/app/__init__.py` — Flask Application Factory

## Overview

This file is the **application factory** for the MiroFish backend. It defines the `create_app()` function which is the standard Flask pattern for creating and configuring the `Flask` application object. Using a factory instead of a module-level `app` variable makes testing and multiple configurations easier.

Key responsibilities:
- Configure the Flask app from the `Config` class.
- Suppress noisy third-party warnings (resource_tracker from transformers).
- Configure JSON encoding to output Chinese characters directly (not as `\uXXXX` escape sequences).
- Set up the logging system.
- Enable CORS for all `/api/*` routes.
- Register a cleanup hook to terminate simulation subprocesses on server shutdown.
- Register the three Blueprint groups (`graph_bp`, `simulation_bp`, `report_bp`) under their respective URL prefixes.
- Provide a `/health` endpoint.
- Add request/response logging middleware.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `Config` | [`config.py`](config.md) | Application configuration |
| `setup_logger`, `get_logger` | [`utils/logger.py`](utils/logger.md) | Structured logging |
| `SimulationRunner` | [`services/simulation_runner.py`](services/simulation_runner.md) | Register cleanup on exit |
| `graph_bp`, `simulation_bp`, `report_bp` | [`api/__init__.py`](api/__init__.md) | Flask Blueprints |

---

## Function Reference

### `create_app(config_class=Config)`

```python
def create_app(config_class=Config) -> Flask:
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `config_class` | `class` | `Config` | The configuration class to load via `app.config.from_object()`. Can be swapped for a testing config. |

**Returns:** A fully initialized `Flask` application instance.

**Step-by-step behaviour:**

1. **Create Flask app** — `Flask(__name__)` and `app.config.from_object(config_class)`.
2. **Disable ASCII escape** — `app.json.ensure_ascii = False` (Flask ≥ 2.3) ensures Chinese text is returned as-is in JSON responses rather than `\u4e2d\u6587`.
3. **Set up logger** — calls `setup_logger('mirofish')`. Startup banners are only printed once (never twice in debug reload mode) by checking the `WERKZEUG_RUN_MAIN` environment variable.
4. **Enable CORS** — `CORS(app, resources={r"/api/*": {"origins": "*"}})` allows the Vue frontend (on a different port) to call the API.
5. **Register cleanup** — `SimulationRunner.register_cleanup()` installs an `atexit` handler that kills all child simulation processes when the Flask process exits.
6. **Request/Response middleware** — `@app.before_request` and `@app.after_request` log every HTTP request/response at DEBUG level.
7. **Register Blueprints:**
   - `graph_bp` → `/api/graph`
   - `simulation_bp` → `/api/simulation`
   - `report_bp` → `/api/report`
8. **Health check** — `GET /health` returns `{"status": "ok", "service": "MiroFish Backend"}`.

---

## URL Map Summary

| Blueprint | Prefix | File |
|-----------|--------|------|
| `graph_bp` | `/api/graph` | [`api/graph.py`](api/graph.md) |
| `simulation_bp` | `/api/simulation` | [`api/simulation.py`](api/simulation.md) |
| `report_bp` | `/api/report` | [`api/report.py`](api/report.md) |
| *(inline)* | `/health` | this file |

---

## Related Files

- [`backend/run.py`](../run.md) — calls `create_app()` and starts the server
- [`backend/app/config.py`](config.md) — supplies the `Config` class
- [`backend/app/api/__init__.py`](api/__init__.md) — declares the three Blueprints
- [`backend/app/services/simulation_runner.py`](services/simulation_runner.md) — `register_cleanup()`
