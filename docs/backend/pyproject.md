# `backend/pyproject.toml` — Backend Python Project Definition

## Overview

`backend/pyproject.toml` is the canonical Python package definition for the backend. It declares metadata, runtime dependencies, development dependencies, build-system settings, and wheel packaging rules.

When the repository uses `uv`, this file is the source of truth for backend dependency resolution.

---

## `[project]`

### Metadata
- `name = "mirofish-backend"`
- `version = "0.1.0"`
- `description`
- `requires-python = ">=3.11"`
- `license = { text = "AGPL-3.0" }`
- `authors = [{ name = "MiroFish Team" }]`

### Runtime dependencies

The dependency list is grouped by purpose in comments.

#### Core framework
- `flask`
- `flask-cors`

#### LLM layer
- `openai`

#### Graph memory
- `zep-cloud`

#### Simulation stack
- `camel-oasis`
- `camel-ai`

#### File parsing
- `PyMuPDF`
- `charset-normalizer`
- `chardet`

#### Utilities
- `python-dotenv`
- `pydantic`

---

## `[project.optional-dependencies]`

### `dev`
Adds development-only tools such as:
- `pytest`
- `pytest-asyncio`
- `pipreqs`

---

## `[build-system]`

Uses:
- `hatchling` as the build backend

This is what allows the backend to be built as a Python package or wheel.

---

## `[dependency-groups]`

Defines a `dev` group for tooling, matching the project’s use of modern dependency managers.

---

## `[tool.hatch.build.targets.wheel]`

Packages the `app` directory when building wheels.

---

## Related Files

- [`requirements.txt`](requirements.md)
- [`../package.json`](../package.md)
- [`../app/config.py`](app/config.md)
