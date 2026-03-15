# `frontend/src/App.vue` — Root Shell

## Overview

`App.vue` is intentionally minimal. It delegates all page rendering to Vue Router and defines a small set of global CSS rules shared by the entire frontend.

---

## Template

### `<router-view />`

This is the placeholder where the active route component is rendered.

That means every screen in the application flows through this root component, but the actual UI logic lives in the route views.

---

## Script

The `<script setup>` block is empty except for a comment. No runtime logic is implemented here.

---

## Styles

The global style block provides:
- CSS reset for margins, padding, and box sizing
- app-wide font stack
- light theme defaults for text and background
- custom scrollbar styling
- inherited button font behavior

These styles affect all routed screens because they are defined at the application root.

---

## Related Files

- [`main.js`](main.md) mounts this component
- [`router/index.js`](router/index.md) determines which view appears inside `<router-view>`
