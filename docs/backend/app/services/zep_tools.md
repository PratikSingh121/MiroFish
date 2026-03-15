# `backend/app/services/zep_tools.py` — Zep Tools Service

## Overview

`zep_tools.py` is the retrieval layer used by the report agent. It wraps raw Zep APIs and exposes higher-level tools that are easier for an LLM-driven agent to use.

The file contains three kinds of code:
- data containers that normalize search, node, edge, and interview results
- low-level graph access helpers
- high-level report tools: `insight_forge`, `panorama_search`, `quick_search`, and `interview_agents`

---

## Result Data Classes

### `SearchResult`

Represents the output of a search query.

Fields:
- `facts`
- `edges`
- `nodes`
- `query`
- `total_count`

Methods:
- `to_dict()` for JSON-like serialization
- `to_text()` for LLM-readable prose summarizing facts

### `NodeInfo`

Normalized view of a Zep node.

Fields:
- `uuid`
- `name`
- `labels`
- `summary`
- `attributes`

Methods:
- `to_dict()`
- `to_text()` which highlights the node name, inferred type, and summary

### `EdgeInfo`

Normalized view of a graph edge with temporal metadata.

Fields include:
- edge identity and fact text
- source and target node UUIDs
- optional source and target names
- optional `created_at`, `valid_at`, `invalid_at`, `expired_at`

Methods and properties:
- `to_dict()`
- `to_text(include_temporal=False)`
- `is_expired`
- `is_invalid`

### `InsightForgeResult`

Aggregates the output of the deep retrieval tool.

It stores:
- original query and simulation requirement
- generated sub-queries
- semantic facts
- entity insights
- relationship chains
- totals for facts, entities, and relationships

Methods:
- `to_dict()`
- `to_text()` for rich LLM-facing analysis text

### `PanoramaResult`

Wide-angle view of a scenario.

It stores:
- all matching nodes
- all edges
- active facts
- historical facts
- totals for nodes, edges, and active/historical counts

Methods:
- `to_dict()`
- `to_text()` that separates current and historical information

### `AgentInterview`

One interviewed agent’s answer block.

Methods:
- `to_dict()`
- `to_text()` which formats role, bio, question, answer, and cleaned quote snippets

### `InterviewResult`

Complete output of a multi-agent interview session.

Stores:
- topic
- question list
- selected agents
- interview records
- selection reasoning
- summary
- counts

Methods:
- `to_dict()`
- `to_text()` that formats the whole interview report for the LLM

---

## `ZepToolsService`

### `__init__(api_key=None, llm_client=None)`

Creates the Zep client and stores an optional LLM client for lazy use in sub-query generation and interview planning.

### `llm` property

Lazy-initializes `LLMClient` only when a tool actually needs it.

### `_call_with_retry(func, operation_name, max_retries=None)`

Retries arbitrary Zep operations with exponential backoff.

---

## Basic Graph Access Methods

### `search_graph(graph_id, query, limit=10, scope='edges') -> SearchResult`

Primary semantic search entry point.

Behavior:
- tries the Zep search API first
- parses matching edges and nodes into a normalized `SearchResult`
- falls back to `_local_search()` if Zep search is unavailable

### `_local_search(graph_id, query, limit=10, scope='edges') -> SearchResult`

Fallback keyword search.

It retrieves all nodes and/or edges locally, scores them against the query, sorts by score, and returns the best matches.

### `get_all_nodes(graph_id) -> List[NodeInfo]`

Fetches every node using [`utils/zep_paging.py`](../utils/zep_paging.md) and converts raw SDK objects to `NodeInfo`.

### `get_all_edges(graph_id, include_temporal=True) -> List[EdgeInfo]`

Fetches every edge with optional temporal fields.

### `get_node_detail(node_uuid) -> Optional[NodeInfo]`

Reads one node directly from Zep.

### `get_node_edges(graph_id, node_uuid) -> List[EdgeInfo]`

Gets all edges in the graph and filters those connected to the target node.

### `get_entities_by_type(graph_id, entity_type) -> List[NodeInfo]`

Filters all nodes by label.

### `get_entity_summary(graph_id, entity_name) -> Dict[str, Any]`

Combines direct entity lookup, semantic search, and node-edge lookup into a compact summary package.

### `get_graph_statistics(graph_id) -> Dict[str, Any]`

Computes counts of nodes, edges, entity types, and relation types.

### `get_simulation_context(graph_id, simulation_requirement, limit=30) -> Dict[str, Any]`

Builds the context package used by report planning:
- related facts
- graph statistics
- filtered entities
- total entity count

---

## High-Level Report Tools

### `insight_forge(graph_id, query, simulation_requirement, report_context='', max_sub_queries=5) -> InsightForgeResult`

Deep retrieval pipeline.

Steps:
1. call `_generate_sub_queries()` to break a question into smaller retrieval targets
2. run semantic edge search for each sub-query and the original query
3. deduplicate facts
4. extract related entity UUIDs from matching edges
5. load those entities with `get_node_detail()`
6. build relationship chains from the matched edge set

This is the strongest tool in the retrieval layer because it combines decomposition, search, entity enrichment, and relationship tracing.

### `_generate_sub_queries(query, simulation_requirement, report_context='', max_queries=5) -> List[str]`

Uses the LLM to turn a broad question into several retrieval-friendly sub-questions.

Falls back to a few generic variants of the original question if JSON generation fails.

### `panorama_search(graph_id, query, include_expired=True, limit=50) -> PanoramaResult`

Wide retrieval tool.

Behavior:
- loads all nodes and all temporal edges
- separates active facts from historical or expired facts
- ranks both sets by relevance to the query
- returns a global picture of the scenario

### `quick_search(graph_id, query, limit=10) -> SearchResult`

Thin wrapper around `search_graph()` for fast, simple lookups.

### `interview_agents(simulation_id, interview_requirement, simulation_requirement='', max_agents=5, custom_questions=None) -> InterviewResult`

Bridges from graph analysis to the running simulation.

Workflow:
1. load simulation profile files with `_load_agent_profiles()`
2. choose likely interview candidates with `_select_agents_for_interview()`
3. generate questions with `_generate_interview_questions()` if the caller did not provide them
4. call `SimulationRunner.interview_agents_batch(...)` against the live OASIS environment
5. clean and merge Twitter and Reddit answers
6. summarize the results with `_generate_interview_summary()`

### `_clean_tool_call_response(response) -> str`

Removes accidental tool-call JSON wrappers from agent responses and extracts the actual text answer.

### `_load_agent_profiles(simulation_id) -> List[Dict[str, Any]]`

Reads `reddit_profiles.json` first, then falls back to `twitter_profiles.csv`, normalizing both into one profile shape.

### `_select_agents_for_interview(profiles, interview_requirement, simulation_requirement, max_agents) -> tuple`

Uses the LLM to choose the most relevant agents for an interview, returning selected profile objects, their indices, and reasoning text.

### `_generate_interview_questions(interview_requirement, simulation_requirement, selected_agents) -> List[str]`

Uses the LLM to create 3–5 open-ended interview questions tailored to the chosen agents and scenario.

### `_generate_interview_summary(interviews, interview_requirement) -> str`

Condenses multi-agent answers into one editorial summary that can be quoted in the final report.

---

## Related Files

- [`services/report_agent.py`](report_agent.md) is the main consumer of this service
- [`utils/zep_paging.py`](../utils/zep_paging.md) powers full-graph traversal
- [`services/simulation_runner.py`](simulation_runner.md) provides the live interview entry point used by `interview_agents()`
