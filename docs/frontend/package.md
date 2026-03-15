# `frontend/package.json` — Frontend Application Package

## Overview

`frontend/package.json` defines the Vue application’s Node package metadata, runtime dependencies, and frontend-specific scripts.

Unlike the root [`package.json`](../package.md), this file describes the actual browser application rather than workspace orchestration.

---

## Metadata

- `name`: `mirofish-frontend`
- `private`: `true`
- `version`: `0.0.0`
- `type`: `module`

### Why `type: module` matters

It enables modern ES module syntax in Vite config and build tooling.

---

## Scripts

### `dev`

Starts the Vite development server.

### `build`

Builds the production frontend bundle.

### `preview`

Serves the production build locally for verification.

---

## Runtime Dependencies

### `vue`
Provides the component framework.

### `vue-router`
Provides route-based navigation between the workflow pages.

### `axios`
Used by the API wrapper layer to call backend endpoints.

### `d3`
Used for graph visualization and interactive rendering.

---

## Development Dependencies

### `vite`
The frontend bundler and dev server.

### `@vitejs/plugin-vue`
Adds Vue single-file component support to Vite.

---

## Related Files

- [`vite.config.js`](vite.config.md)
- [`src/main.js`](src/main.md)
- [`src/router/index.js`](src/router/index.md)
- [`../package.json`](../package.md)
