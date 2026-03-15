# `Dockerfile` — Development Container Image

## Overview

The root `Dockerfile` defines a single container image that can run both the MiroFish frontend and backend in development mode. It combines:
- Python 3.11 for the Flask backend
- Node.js/npm for the Vite frontend and root scripts
- `uv` for Python dependency installation and execution

The final command starts both services together through the root npm `dev` script.

---

## Build Steps

### `FROM python:3.11`

Uses the official Python 3.11 image as the base.

### `RUN apt-get update ... apt-get install -y nodejs npm`

Installs Node.js and npm into the Python image so both runtimes are available in one container.

### `COPY --from=ghcr.io/astral-sh/uv:0.9.26 /uv /uvx /bin/`

Copies the `uv` and `uvx` executables from the official Astral image instead of installing them with a shell script.

### `WORKDIR /app`

Sets `/app` as the working directory for the rest of the build and runtime steps.

### Dependency-cache layer copies

The Dockerfile copies only dependency manifests first:
- root `package.json` and `package-lock.json`
- `frontend/package.json` and `frontend/package-lock.json`
- `backend/pyproject.toml` and `backend/uv.lock`

This allows Docker to reuse cached dependency layers when source files change but lockfiles do not.

### `RUN npm ci && npm ci --prefix frontend && cd backend && uv sync --frozen`

Installs:
- root Node dependencies
- frontend Node dependencies
- backend Python dependencies from the `uv.lock` lockfile

### `COPY . .`

Copies the rest of the repository into the image after dependencies are already installed.

### `EXPOSE 3000 5001`

Declares the frontend and backend ports.

### `CMD ["npm", "run", "dev"]`

Starts the root `dev` script, which launches both backend and frontend concurrently.

---

## Runtime Outcome

Inside this container, the default process is equivalent to running:

```bash
npm run dev
```

which in turn starts:
- the backend on port `5001`
- the frontend on port `3000`

---

## Related Files

- [`docker-compose.yml`](docker-compose.md)
- [`package.json`](package.md)
- [`backend/pyproject.toml`](backend/pyproject.md)
- [`frontend/package.json`](frontend/package.md)
