# `frontend/src/views/Home.vue` — Landing Page

## Overview

`Home.vue` is the entry screen where the user defines the simulation problem and uploads source documents. It also embeds the history browser so prior runs can be revisited.

This page does not immediately call the backend. Instead, it stores the selected files and requirement in [`store/pendingUpload.js`](../store/pendingUpload.md) and navigates to the process page, where the upload actually begins.

---

## State

Main refs:
- `formData` for project name and simulation requirement
- `files`
- `loading`
- `error`
- `isDragOver`
- `fileInput`

### Computed

- `canSubmit` enables the main action only when files and a requirement are present

---

## Functions

### `triggerFileInput()`
Programmatically opens the hidden file input.

### `handleFileSelect(event)`
Reads files chosen through the file picker.

### `handleDragOver(e)`
Sets drag-over UI state.

### `handleDragLeave(e)`
Clears drag-over state.

### `handleDrop(e)`
Handles drag-and-drop file ingestion.

### `addFiles(newFiles)`
Appends new files to the local list.

### `removeFile(index)`
Removes one selected file.

### `scrollToBottom()`
Scroll helper used after file additions or UI changes.

### `startSimulation()`
Main submit action.

Behavior:
1. validates input
2. writes files and requirement into the pending-upload store
3. navigates to the process route for actual backend execution

---

## Child Components

- [`components/HistoryDatabase.vue`](../components/HistoryDatabase.md) shows previous simulations

---

## Related Files

- [`store/pendingUpload.js`](../store/pendingUpload.md)
- [`views/MainView.vue`](MainView.md)
