# `frontend/src/main.js` — Vue Entry Point

## Overview

`main.js` is the bootstrap file for the frontend. It creates the Vue application, installs the router, and mounts the app into the `#app` element from `frontend/index.html`.

---

## Code Flow

### `createApp(App)`

Creates the root Vue application using [`App.vue`](App.md) as the root component.

### `app.use(router)`

Registers the router from [`router/index.js`](router/index.md), enabling route-based navigation across the five workflow steps.

### `app.mount('#app')`

Mounts the app into the DOM.

---

## Related Files

- [`App.vue`](App.md) is the root component
- [`router/index.js`](router/index.md) defines page navigation
