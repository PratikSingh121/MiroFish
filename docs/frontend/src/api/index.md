# `frontend/src/api/index.js` — API Client

## Overview

`api/index.js` centralizes HTTP configuration for the frontend. It creates one Axios instance, sets timeouts and base URL defaults, and adds a small retry helper used by higher-level API modules.

---

## Axios Instance

### `service`

Configured with:
- `baseURL = import.meta.env.VITE_API_BASE_URL || 'http://localhost:5001'`
- `timeout = 300000` (5 minutes)
- default `Content-Type: application/json`

The long timeout is important because ontology generation and report generation can take a while.

---

## Interceptors

### Request interceptor

Currently passes requests through unchanged, but provides one place for future auth headers or request instrumentation.

### Response interceptor

Behavior:
- unwraps `response.data`
- treats `{ success: false }` style API payloads as errors
- logs timeout and network failures
- rejects Promise chains with meaningful errors

This matches the backend’s convention of returning JSON envelopes with a `success` field.

---

## `requestWithRetry(requestFn, maxRetries = 3, delay = 1000)`

Generic exponential-retry helper for UI requests.

Behavior:
1. runs the provided async request function
2. retries up to `maxRetries`
3. waits `delay * 2^i` between attempts
4. rethrows the final error if all attempts fail

This is used for operations where transient backend or network failures are plausible.

---

## Exports

- default export: `service`
- named export: `requestWithRetry`

---

## Related Files

- [`api/graph.js`](graph.md)
- [`api/simulation.js`](simulation.md)
- [`api/report.js`](report.md)
