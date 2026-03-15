# `backend/app/services/graph_builder.py` — Graph Construction Service

## Overview

`graph_builder.py` is responsible for constructing the **Zep knowledge graph** from the document text and ontology. It communicates directly with the Zep Cloud API to:

1. Create a new standalone graph.
2. Set the ontology schema (entity types and edge types) so Zep knows what to extract.
3. Feed the document text in batches as "episodes" (Zep's unit of ingestion).
4. Wait for Zep to asynchronously process and extract the knowledge graph from the episodes.
5. Return graph statistics (node count, edge count, entity types).

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `Zep` | `zep_cloud` SDK | Zep Cloud API client |
| `EpisodeData`, `EntityEdgeSourceTarget` | `zep_cloud` | API data structures |
| `TaskManager`, `TaskStatus` | [`models/task.py`](../models/task.md) | Progress reporting |
| `TextProcessor` | [`services/text_processor.py`](text_processor.md) | Text chunking |
| `fetch_all_nodes`, `fetch_all_edges` | [`utils/zep_paging.py`](../utils/zep_paging.md) | Paginated node/edge retrieval |
| `Config` | [`config.py`](../config.md) | `ZEP_API_KEY` |

---

## `GraphInfo` Dataclass

```python
@dataclass
class GraphInfo:
    graph_id: str
    node_count: int
    edge_count: int
    entity_types: List[str]
```

Returned by `_get_graph_info()` and stored in the task result dict.

---

## `GraphBuilderService` Class

### `__init__(api_key=None)`

Initialises the Zep client. Raises `ValueError` if `ZEP_API_KEY` is not configured. Also acquires a `TaskManager` singleton for progress reporting.

---

### `build_graph_async(text, ontology, graph_name, chunk_size, chunk_overlap, batch_size) → str`

**The main asynchronous entry point.** Called by [`api/graph.py`](../api/graph.md).

Creates a `TaskManager` task and launches `_build_graph_worker()` in a daemon thread. Returns the `task_id` immediately so the caller can poll progress.

---

### `_build_graph_worker(task_id, text, ontology, graph_name, chunk_size, chunk_overlap, batch_size)`

**Background thread function.** Executes all build steps sequentially and reports progress to `TaskManager` at each stage:

| Progress | Action |
|----------|--------|
| 5% | Start |
| 10% | Graph created |
| 15% | Ontology set |
| 20% | Text chunked |
| 20–60% | Batches uploaded (scales linearly) |
| 60–90% | Waiting for Zep processing |
| 90% | Collecting graph info |
| 100% | Complete |

On exception, calls `task_manager.fail_task()`.

---

### `create_graph(name) → str`

**Public method.** Creates a new named graph in Zep. The graph ID is `"mirofish_" + 16 random hex chars`. Calls `self.client.graph.create(graph_id=..., name=..., description=...)`.

Returns the `graph_id` string.

---

### `set_ontology(graph_id, ontology)`

**Public method.** Converts the ontology dict (from `OntologyGenerator`) into Zep SDK Pydantic model classes and calls `client.graph.set_ontology()`.

**How dynamic class creation works:**

Zep SDK's `set_ontology` expects Pydantic model classes (not instances) as arguments. Since the entity types are dynamically determined by the LLM, MiroFish uses Python's `type()` built-in to create Pydantic-compatible classes at runtime:

```python
entity_class = type(name, (EntityModel,), attrs)
```

This creates a new class named `name` that inherits from `EntityModel` with field definitions built from the ontology's `attributes` list.

**Reserved attribute names** (`uuid`, `name`, `group_id`, `name_embedding`, `summary`, `created_at`) are prefixed with `entity_` to avoid conflicts with Zep's internal fields.

---

### `add_text_batches(graph_id, chunks, batch_size, progress_callback) → List[str]`

**Public method.** Sends text chunks to Zep in batches. Each batch creates `EpisodeData` objects and calls `client.graph.add(graph_id=..., episodes=[...])`. Returns a list of episode UUIDs for tracking.

Calls `progress_callback(message, progress_fraction)` after each batch.

---

### `_wait_for_episodes(episode_uuids, progress_callback)`

**Private method.** Polls Zep until all submitted episodes have been processed (status changes from `"pending"`) to `"processed"` or similar). Uses exponential backoff between checks. Calls `progress_callback` during the wait.

---

### `_get_graph_info(graph_id) → GraphInfo`

**Private method.** After processing is complete, fetches all nodes and edges using the paginated utilities `fetch_all_nodes()` and `fetch_all_edges()`. Extracts the unique entity types from node labels and returns a `GraphInfo` object.

---

## Related Files

- [`api/graph.py`](../api/graph.md) — calls `build_graph_async()` and `create_graph()`, `set_ontology()` directly
- [`models/task.py`](../models/task.md) — task progress tracking
- [`services/text_processor.py`](text_processor.md) — `split_text()` used for chunking
- [`utils/zep_paging.py`](../utils/zep_paging.md) — paginated node/edge fetching
- [`services/ontology_generator.py`](ontology_generator.md) — produces the ontology dict consumed here
