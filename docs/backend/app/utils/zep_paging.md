# `backend/app/utils/zep_paging.py` — Zep Pagination

## Overview

`zep_paging.py` hides the UUID-cursor pagination used by the Zep Cloud node and edge list APIs. Callers ask for “all nodes” or “all edges”; this module handles page traversal, retry logic, and safety limits.

This keeps higher-level services like [`services/zep_tools.py`](../services/zep_tools.md) simple.

---

## Constants

| Constant | Purpose |
|----------|---------|
| `_DEFAULT_PAGE_SIZE` | default Zep page size |
| `_MAX_NODES` | safety cap for node retrieval |
| `_DEFAULT_MAX_RETRIES` | per-page retry count |
| `_DEFAULT_RETRY_DELAY` | initial retry delay for page fetches |

---

## `_fetch_page_with_retry(api_call, *args, ..., **kwargs) -> list[Any]`

Low-level helper for one paged API call.

Behavior:
1. validates `max_retries >= 1`
2. calls the provided API method
3. retries only transient-style failures such as `ConnectionError`, `TimeoutError`, `OSError`, and `InternalServerError`
4. doubles the retry delay after each failure
5. logs warnings for retry attempts and an error for the final failure

This function is the core building block used by both exported pagination helpers.

---

## `fetch_all_nodes(client, graph_id, page_size=100, max_items=2000, ...) -> list[Any]`

Fetches node pages until one of these stopping conditions is met:
- Zep returns an empty page
- returned page size is smaller than `page_size`
- node count reaches `max_items`
- the final node in a page has no UUID cursor field

The function tracks the next UUID cursor using either `uuid_` or `uuid` depending on the SDK object shape.

### Why the `max_items` cap exists

Node retrieval is capped by default at 2000 entries to prevent very large graphs from blowing up memory usage or request cost.

---

## `fetch_all_edges(client, graph_id, page_size=100, ...) -> list[Any]`

Same cursor-driven loop as `fetch_all_nodes()`, but without the explicit node-count cap.

Stops when:
- the current page is empty
- the current page is shorter than `page_size`
- the last edge has no usable UUID field for the next cursor

---

## Related Files

- [`services/zep_tools.py`](../services/zep_tools.md) uses both helpers extensively
- [`services/zep_entity_reader.py`](../services/zep_entity_reader.md) depends on complete graph traversal behavior
