# `package.json` — Root Workspace Scripts

## Overview

The root `package.json` is the workspace-level control file for the whole repository. It does not describe the frontend app itself; instead, it coordinates:
- top-level setup tasks
- backend startup
- frontend startup
- concurrent development execution
- production frontend build delegation

---

## Metadata

- `name`: `mirofish`
- `version`: `0.1.0`
- `description`: short repository description
- `license`: `AGPL-3.0`

### `engines`

Requires Node.js `>=18.0.0`.

---

## Scripts

### `setup`

Runs:

```bash
npm install && cd frontend && npm install
```

Installs root and frontend Node dependencies.

### `setup:backend`

Runs:

```bash
cd backend && uv sync
```

Installs Python dependencies in the backend using `uv`.

### `setup:all`

Runs both Node and backend setup.

### `dev`

Runs backend and frontend together through `concurrently`:
- backend process name: `backend`
- frontend process name: `frontend`

If either process exits, `--kill-others` stops the other one.

### `backend`

Starts the Flask backend via:

```bash
cd backend && uv run python run.py
```

### `frontend`

Starts the Vite dev server in the frontend directory.

### `build`

Delegates the production build to the frontend app.

---

## `devDependencies`

### `concurrently`

Used only at the workspace level to run the frontend and backend together in development.

---

## Related Files

- [`frontend/package.json`](frontend/package.md)
- [`backend/pyproject.toml`](backend/pyproject.md)
- [`Dockerfile`](Dockerfile.md)
- [`repo-readme.md`](repo-readme.md)
