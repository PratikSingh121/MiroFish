# `frontend/src/store/pendingUpload.js` — Pending Upload Store

## Overview

`pendingUpload.js` is a tiny reactive store used to bridge the gap between the home page and the process page.

The home page collects uploaded files and the simulation requirement, but the actual API call happens after navigation. This store temporarily keeps that data in memory until the process view consumes it.

---

## State

Reactive fields:
- `files`
- `simulationRequirement`
- `isPending`

---

## Functions

### `setPendingUpload(files, requirement)`

Stores the file list and requirement and marks the store as pending.

### `getPendingUpload()`

Returns a plain object snapshot of the current reactive state.

### `clearPendingUpload()`

Resets the store after the process page has consumed the pending upload.

---

## Default Export

The reactive `state` object itself is also exported as default.

---

## Related Files

- [`views/Home.vue`](../views/Home.md) writes into this store
- [`views/MainView.vue`](../views/MainView.md) reads and clears it when creating a new project
