# `backend/run.py` — Application Entry Point

## Overview

`run.py` is the **top-level launch script** for the MiroFish Flask backend. It is the file you execute directly (`python run.py`) to start the development server. Its responsibilities are:

1. **Fix Windows console encoding** — ensures UTF-8 output so that Chinese characters appear correctly in the terminal on Windows systems.
2. **Validate required configuration** — calls `Config.validate()` and exits early with a helpful error message if critical environment variables (like `LLM_API_KEY` or `ZEP_API_KEY`) are missing.
3. **Create and launch the Flask application** — calls the application factory `create_app()` and starts the server.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `create_app` | [`backend/app/__init__.py`](../app/__init__.md) | Flask application factory |
| `Config` | [`backend/app/config.py`](../app/config.md) | Reads `.env` and exposes all configuration values |

---

## Function Reference

### `main()`

```python
def main():
```

**What it does:**

1. Calls `Config.validate()` to check that `LLM_API_KEY` and `ZEP_API_KEY` are set. If there are errors, it prints them and exits with code `1`.
2. Calls `create_app()` (the Flask application factory) to build the initialized Flask instance.
3. Reads `FLASK_HOST` (default `0.0.0.0`), `FLASK_PORT` (default `5001`), and `Config.DEBUG` from the environment.
4. Calls `app.run(...)` with `threaded=True` so concurrent HTTP requests are handled in separate threads.

**Environment variables read:**

| Variable | Default | Meaning |
|----------|---------|---------|
| `FLASK_HOST` | `0.0.0.0` | Network interface to bind to |
| `FLASK_PORT` | `5001` | TCP port |
| `FLASK_DEBUG` | `True` | Enable Flask debug mode & reloader |

---

## Windows UTF-8 Encoding Fix

Before any imports, `run.py` checks `sys.platform == 'win32'` and reconfigures `sys.stdout` and `sys.stderr` to UTF-8. This is important because OASIS simulation logs and LLM responses frequently contain Chinese text which would otherwise display as mojibake on Windows.

```python
if sys.platform == 'win32':
    os.environ.setdefault('PYTHONIOENCODING', 'utf-8')
    if hasattr(sys.stdout, 'reconfigure'):
        sys.stdout.reconfigure(encoding='utf-8', errors='replace')
```

---

## Entry Guard

```python
if __name__ == '__main__':
    main()
```

The file only executes `main()` when run directly, not when imported (e.g. by Gunicorn or pytest).

---

## Related Files

- [`backend/app/__init__.py`](../app/__init__.md) — Flask app factory (`create_app`)
- [`backend/app/config.py`](../app/config.md) — `Config` class with all environment variables
