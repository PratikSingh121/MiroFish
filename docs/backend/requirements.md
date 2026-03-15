# `backend/requirements.txt` — Backend pip Requirements

## Overview

`backend/requirements.txt` is the pip-style dependency list for the backend. It overlaps heavily with [`pyproject.toml`](pyproject.md), but exists to support environments or tooling that still expect a classic requirements file.

---

## Dependency Groups

The file is organized with comment headings.

### Core framework
- `flask`
- `flask-cors`

### LLM
- `openai`

### Zep Cloud
- `zep-cloud`

### OASIS simulation
- `camel-oasis`
- `camel-ai`

### File processing
- `PyMuPDF`
- `charset-normalizer`
- `chardet`

### Utility libraries
- `python-dotenv`
- `pydantic`

---

## Why Both Files Exist

- `pyproject.toml` is the modern canonical definition used by `uv`
- `requirements.txt` is a compatibility artifact for plain pip-based installs and external tooling

---

## Related Files

- [`pyproject.toml`](pyproject.md)
- [`../app/utils/file_parser.py`](app/utils/file_parser.md)
- [`../app/utils/llm_client.py`](app/utils/llm_client.md)
