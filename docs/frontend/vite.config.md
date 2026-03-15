# `frontend/vite.config.js` — Frontend Dev Server Configuration

## Overview

`vite.config.js` configures the frontend build tool and development server. Its job is to make local development smooth by:
- enabling Vue support
- serving the app on a fixed port
- opening the browser automatically
- proxying API traffic to the Flask backend

---

## `defineConfig(...)`

The file exports a Vite configuration object through `defineConfig`.

### `plugins: [vue()]`

Enables Vue single-file component compilation.

### `server.port = 3000`

Pins the frontend dev server to port `3000`.

This matters because repository docs, Docker config, and the root scripts all assume that port.

### `server.open = true`

Automatically opens a browser window when the dev server starts.

### `server.proxy['/api']`

Proxies frontend requests beginning with `/api` to:

```text
http://localhost:5001
```

This removes the need for the frontend to hardcode cross-origin backend URLs during development.

### `server.proxy['/api'].changeOrigin = true`

Makes the proxied request appear as if it originated from the target host.

---

## Why This File Matters

Without this proxy setup, the browser frontend would need separate backend host handling and would be more likely to hit CORS and environment mismatch problems during development.

---

## Related Files

- [`package.json`](package.md)
- [`src/api/index.js`](src/api/index.md)
- [`../docker-compose.yml`](../docker-compose.md)
- [`../Dockerfile`](../Dockerfile.md)
