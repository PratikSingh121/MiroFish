# `backend/app/config.py` — Application Configuration

## Overview

`config.py` is the **single source of truth** for all runtime configuration in the MiroFish backend. It uses `python-dotenv` to load a `.env` file from the project root directory and then exposes every setting as a class attribute on the `Config` class.

The module-level code performs the `.env` lookup before the class is defined, so the environment variables are available at import time.

---

## `.env` File Loading

```python
project_root_env = os.path.join(os.path.dirname(__file__), '../../.env')

if os.path.exists(project_root_env):
    load_dotenv(project_root_env, override=True)
else:
    load_dotenv(override=True)
```

The path calculation resolves to `MiroFish/.env` (two levels above `backend/app/config.py`). If that file does not exist (e.g. in a Docker container where variables are injected directly), `load_dotenv(override=True)` reads whatever is already in `os.environ`.

---

## `Config` Class

All attributes are **class-level**, not instance-level. They are read once at import time from `os.environ`.

### Flask Core Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `SECRET_KEY` | `SECRET_KEY` | `'mirofish-secret-key'` | Flask session signing key |
| `DEBUG` | `FLASK_DEBUG` | `True` | Enable Flask debug mode and auto-reloader |
| `JSON_AS_ASCII` | *(hardcoded)* | `False` | Disable ASCII escaping in JSON responses |

### LLM Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `LLM_API_KEY` | `LLM_API_KEY` | `None` | API key for any OpenAI-compatible LLM provider |
| `LLM_BASE_URL` | `LLM_BASE_URL` | `'https://api.openai.com/v1'` | Base URL for the LLM API endpoint |
| `LLM_MODEL_NAME` | `LLM_MODEL_NAME` | `'gpt-4o-mini'` | Model identifier to use for all LLM calls |

### Zep Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `ZEP_API_KEY` | `ZEP_API_KEY` | `None` | API key for Zep Cloud (graph memory service) |

### File Upload Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `MAX_CONTENT_LENGTH` | *(hardcoded)* | `50 * 1024 * 1024` | Maximum request body size (50 MB) |
| `UPLOAD_FOLDER` | *(hardcoded)* | `backend/uploads/` | Root directory for all uploaded files and project data |
| `ALLOWED_EXTENSIONS` | *(hardcoded)* | `{'pdf','md','txt','markdown'}` | Accepted file types for document upload |

### Text Processing Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `DEFAULT_CHUNK_SIZE` | *(hardcoded)* | `500` | Default character count per text chunk when splitting |
| `DEFAULT_CHUNK_OVERLAP` | *(hardcoded)* | `50` | Number of characters to overlap between consecutive chunks |

### OASIS Simulation Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `OASIS_DEFAULT_MAX_ROUNDS` | `OASIS_DEFAULT_MAX_ROUNDS` | `10` | Default number of simulation rounds |
| `OASIS_SIMULATION_DATA_DIR` | *(hardcoded)* | `backend/uploads/simulations/` | Directory where per-simulation data is stored |
| `OASIS_TWITTER_ACTIONS` | *(hardcoded)* | `[...]` | Valid action types for Twitter platform agents |
| `OASIS_REDDIT_ACTIONS` | *(hardcoded)* | `[...]` | Valid action types for Reddit platform agents |

**OASIS Twitter Actions:** `CREATE_POST`, `LIKE_POST`, `REPOST`, `FOLLOW`, `DO_NOTHING`, `QUOTE_POST`

**OASIS Reddit Actions:** `LIKE_POST`, `DISLIKE_POST`, `CREATE_POST`, `CREATE_COMMENT`, `LIKE_COMMENT`, `DISLIKE_COMMENT`, `SEARCH_POSTS`, `SEARCH_USER`, `TREND`, `REFRESH`, `DO_NOTHING`, `FOLLOW`, `MUTE`

### Report Agent Settings

| Attribute | Env Variable | Default | Description |
|-----------|-------------|---------|-------------|
| `REPORT_AGENT_MAX_TOOL_CALLS` | `REPORT_AGENT_MAX_TOOL_CALLS` | `5` | Maximum tool calls per ReACT iteration |
| `REPORT_AGENT_MAX_REFLECTION_ROUNDS` | `REPORT_AGENT_MAX_REFLECTION_ROUNDS` | `2` | Number of reflection passes per report section |
| `REPORT_AGENT_TEMPERATURE` | `REPORT_AGENT_TEMPERATURE` | `0.5` | LLM temperature for report generation |

---

## Methods

### `Config.validate()`

```python
@classmethod
def validate(cls) -> list[str]:
```

Checks that all *required* configuration values are set. Currently validates:
- `LLM_API_KEY` — must not be `None`
- `ZEP_API_KEY` — must not be `None`

Returns a list of error message strings. An empty list means all required config is present. See [`backend/run.py`](../run.md) for how validation errors are handled at startup.

---

## Related Files

- [`backend/run.py`](../run.md) — calls `Config.validate()` before starting
- [`backend/app/__init__.py`](__init__.md) — loads config via `app.config.from_object(Config)`
- [`backend/app/utils/llm_client.py`](utils/llm_client.md) — reads `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME`
- [`backend/app/services/graph_builder.py`](services/graph_builder.md) — reads `ZEP_API_KEY`
