# `docker-compose.yml` — Container Runtime Definition

## Overview

The root `docker-compose.yml` defines a single service named `mirofish` for running the prebuilt container image.

Unlike the `Dockerfile`, which explains how to build an image, this file explains how to run one with the correct ports, environment variables, restart policy, and persistent upload storage.

---

## Service: `mirofish`

### `image: ghcr.io/666ghj/mirofish:latest`

Pulls the published MiroFish image from GitHub Container Registry.

A commented mirror line is also included for users who need a faster registry endpoint.

### `container_name: mirofish`

Gives the running container a stable name.

### `env_file: .env`

Loads runtime environment variables from the repository’s `.env` file.

This is how API keys and model settings are supplied to the containerized backend.

### `ports`

Maps host ports to container ports:
- `3000:3000` for the frontend
- `5001:5001` for the backend API

### `restart: unless-stopped`

Keeps the service running across Docker daemon restarts unless the user explicitly stops it.

### `volumes`

Maps `./backend/uploads` on the host to `/app/backend/uploads` in the container.

This preserves uploaded files, generated reports, and simulation artifacts outside the container filesystem.

---

## Practical Meaning

Running:

```bash
docker compose up -d
```

with this file gives you a persistent, self-contained MiroFish instance backed by the published container image.

---

## Related Files

- [`Dockerfile`](Dockerfile.md)
- [`repo-readme.md`](repo-readme.md)
