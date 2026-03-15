# `backend/app/services/zep_entity_reader.py` — Zep Entity Reader Service

## Overview

`zep_entity_reader.py` is the **read-side adapter for the Zep knowledge graph**. After the graph has been built, this service is responsible for fetching the stored entities (nodes) and their relationships (edges) in a form that the rest of the MiroFish pipeline can use.

Its primary mission is to **filter** the raw Zep graph output to find only the "real" entities — those that match the ontology-defined types — and optionally enrich each entity with its connected edges and neighbouring nodes.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `Zep` | `zep_cloud` SDK | Zep Cloud API client |
| `fetch_all_nodes`, `fetch_all_edges` | [`utils/zep_paging.py`](../utils/zep_paging.md) | Paginated retrieval of all nodes/edges |
| `Config` | [`config.py`](../config.md) | `ZEP_API_KEY` |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Data Structures

### `EntityNode` Dataclass

Represents a single entity node retrieved from Zep.

```python
@dataclass
class EntityNode:
    uuid: str
    name: str
    labels: List[str]        # e.g. ["Entity", "Student"]
    summary: str             # Zep-generated summary of the entity
    attributes: Dict         # Key-value attributes from the ontology
    related_edges: List[Dict]  # Edges connecting this node
    related_nodes: List[Dict]  # Neighbouring entity nodes
```

#### `EntityNode.get_entity_type() → Optional[str]`

Returns the first label that is neither `"Entity"` nor `"Node"`. This is the ontology type (e.g. `"Student"`, `"University"`).

#### `EntityNode.to_dict() → Dict`

Serializes the node to a plain dict for JSON API responses.

---

### `FilteredEntities` Dataclass

The result of a filtering operation:

```python
@dataclass
class FilteredEntities:
    entities: List[EntityNode]
    entity_types: Set[str]  # All distinct types found
    total_count: int        # Total nodes in graph before filtering
    filtered_count: int     # Number of entities after filtering
```

#### `FilteredEntities.to_dict() → Dict`

Converts to JSON-serializable dict.

---

## `ZepEntityReader` Class

### `__init__(api_key=None)`

Initialises the Zep client. Raises `ValueError` if `ZEP_API_KEY` is missing.

---

### `_call_with_retry(func, operation_name, max_retries=3, initial_delay=2.0) → T`

**Generic retry wrapper** for Zep API calls. Uses exponential backoff (each failure doubles the delay). Logs warnings on intermediate failures and an error after all retries are exhausted. Used internally by `get_node_edges()` and `get_entity_with_context()`.

---

### `get_all_nodes(graph_id) → List[Dict]`

Fetches every node in the graph using `fetch_all_nodes()` (which handles pagination). Converts each raw Zep node into a normalised dict with keys: `uuid`, `name`, `labels`, `summary`, `attributes`.

---

### `get_all_edges(graph_id) → List[Dict]`

Fetches every edge in the graph using `fetch_all_edges()`. Returns dicts with: `uuid`, `name`, `fact`, `source_node_uuid`, `target_node_uuid`, `attributes`.

---

### `get_node_edges(node_uuid) → List[Dict]`

**Per-node edge lookup.** Calls `client.graph.node.get_entity_edges(node_uuid=...)` with retry. Returns a list of edge dicts.

This is used for the "enrich" step when building `EntityNode` objects with context.

---

### `filter_defined_entities(graph_id, defined_entity_types=None, enrich_with_edges=True) → FilteredEntities`

**Main public method.** The core filtering logic:

1. Fetches all nodes via `get_all_nodes()`.
2. Fetches all edges via `get_all_edges()` (for building node→neighbours maps).
3. For each node, checks if it has any label that is *not* `"Entity"` or `"Node"`. Only such nodes pass the filter.
4. If `defined_entity_types` is provided, further filters to only include nodes whose type is in that list.
5. If `enrich_with_edges=True`, builds `related_edges` and `related_nodes` for each entity by matching edge endpoints.
6. Returns a `FilteredEntities` object.

---

### `get_entities_by_type(graph_id, entity_type, enrich_with_edges=True) → List[EntityNode]`

Convenience wrapper calling `filter_defined_entities()` with `defined_entity_types=[entity_type]`. Returns just the `entities` list.

---

### `get_entity_with_context(graph_id, entity_uuid) → Optional[EntityNode]`

Looks up a single entity and enriches it with its edges and neighbouring node names. Returns `None` if the entity doesn't exist in the graph.

---

## Design Note: Why Filter?

Zep's graph contains two kinds of nodes:
- **Typed entity nodes** — have labels like `["Entity", "Student"]`. These map directly to the ontology.
- **Generic nodes** — have only `["Entity"]` or `["Node"]` labels. These are intermediate concepts, events, or facts that Zep extracted but don't correspond to a simulation agent.

Only the typed entity nodes are meaningful for agent generation. The filter step ensures that `OasisProfileGenerator` only processes real, identifiable entities.

---

## Related Files

- [`api/simulation.py`](../api/simulation.md) — calls `filter_defined_entities()` and other methods for entity listing endpoints
- [`services/oasis_profile_generator.py`](oasis_profile_generator.md) — consumes `EntityNode` objects to generate agent personas
- [`services/simulation_config_generator.py`](simulation_config_generator.md) — uses entities to build per-agent activity configs
- [`utils/zep_paging.py`](../utils/zep_paging.md) — handles paginated node/edge fetching
