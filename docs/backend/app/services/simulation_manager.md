# `backend/app/services/simulation_manager.py` — Simulation Manager

## Overview

`simulation_manager.py` is the **central orchestrator of the simulation preparation pipeline**. It coordinates three major sub-services — entity reading, profile generation, and config generation — to transform a raw Zep knowledge graph into a ready-to-run OASIS simulation.

The `SimulationManager` owns the lifecycle of a `SimulationState` from creation through preparation to readiness. State is persisted in `uploads/simulations/<sim_id>/state.json` and cached in memory so the API can serve status polls cheaply.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `ZepEntityReader`, `FilteredEntities` | [`services/zep_entity_reader.py`](zep_entity_reader.md) | Reads + filters Zep graph nodes |
| `OasisProfileGenerator`, `OasisAgentProfile` | [`services/oasis_profile_generator.py`](oasis_profile_generator.md) | Generates OASIS agent profiles |
| `SimulationConfigGenerator`, `SimulationParameters` | [`services/simulation_config_generator.py`](simulation_config_generator.md) | LLM-based config generation |
| `Config` | [`app/config.py`](../config.md) | API keys |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Enums

### `SimulationStatus`

The full lifecycle of a simulation:

```
CREATED → PREPARING → READY → RUNNING → PAUSED → STOPPED/COMPLETED/FAILED
```

| Value | Meaning |
|-------|---------|
| `CREATED` | Simulation record exists, no preparation started |
| `PREPARING` | Background thread running preparation workflow |
| `READY` | Config + profiles saved; ready to start OASIS |
| `RUNNING` | OASIS subprocess is live |
| `PAUSED` | Paused mid-simulation |
| `STOPPED` | Manually stopped by user |
| `COMPLETED` | Simulation ran to its natural completion |
| `FAILED` | An error occurred during preparation or running |

### `PlatformType`

```python
class PlatformType(str, Enum):
    TWITTER = "twitter"
    REDDIT  = "reddit"
```

---

## Data Structures

### `SimulationState`

Tracks the full state of one simulation instance.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `simulation_id` | `str` | — | `sim_<12-char hex>` |
| `project_id` | `str` | — | Parent project |
| `graph_id` | `str` | — | Zep graph used |
| `enable_twitter` | `bool` | `True` | Spawn Twitter branch |
| `enable_reddit` | `bool` | `True` | Spawn Reddit branch |
| `status` | `SimulationStatus` | `CREATED` | Current lifecycle stage |
| `entities_count` | `int` | `0` | Typed entities found in graph |
| `profiles_count` | `int` | `0` | OASIS profiles generated |
| `entity_types` | `List[str]` | `[]` | Entity type names used |
| `config_generated` | `bool` | `False` | Whether LLM config succeeded |
| `config_reasoning` | `str` | `""` | LLM's explanation of parameter choices |
| `current_round` | `int` | `0` | Last completed simulation round |
| `twitter_status` | `str` | `"not_started"` | Twitter branch status |
| `reddit_status` | `str` | `"not_started"` | Reddit branch status |
| `created_at` | `ISO str` | `now()` | Creation timestamp |
| `updated_at` | `ISO str` | `now()` | Last write timestamp |
| `error` | `Optional[str]` | `None` | Error message if failed |

#### `SimulationState.to_dict()`

Returns a complete dictionary for file persistence (all fields).

#### `SimulationState.to_simple_dict()`

Returns a condensed dictionary for API responses — only: `simulation_id`, `project_id`, `graph_id`, `status`, `entities_count`, `profiles_count`, `entity_types`, `config_generated`, `error`.

---

## `SimulationManager` Class

### Class Attribute: `SIMULATION_DATA_DIR`

```python
SIMULATION_DATA_DIR = os.path.join(os.path.dirname(__file__), '../../uploads/simulations')
```

Filesystem root where each simulation's directory is stored.

---

### `__init__()`

Creates `uploads/simulations/` if missing. Initialises `self._simulations: Dict[str, SimulationState]` — an in-memory cache to avoid repeated disk reads.

---

### `_get_simulation_dir(simulation_id) → str`

Returns (and creates) `SIMULATION_DATA_DIR/<simulation_id>/`.

---

### `_save_simulation_state(state)`

1. Gets simulation directory.
2. Updates `state.updated_at` to now.
3. Writes `state.to_dict()` to `state.json`.
4. Updates in-memory cache at `self._simulations[state.simulation_id]`.

---

### `_load_simulation_state(simulation_id) → Optional[SimulationState]`

1. Returns cached state if present.
2. Otherwise reads `state.json` from disk.
3. Reconstructs a `SimulationState` dataclass from the JSON dict (handles missing keys with sensible defaults).
4. Populates cache and returns.

Returns `None` if the directory or file does not exist.

---

### `create_simulation(project_id, graph_id, enable_twitter, enable_reddit) → SimulationState`

Creates a new simulation record.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `project_id` | `str` | required | Parent project |
| `graph_id` | `str` | required | Zep graph from which to draw entities |
| `enable_twitter` | `bool` | `True` | Whether Twitter OASIS branch is used |
| `enable_reddit` | `bool` | `True` | Whether Reddit OASIS branch is used |

**Behavior:**
1. Generates `simulation_id = "sim_" + 12-hex UUID.
2. Creates `SimulationState(status=CREATED)`.
3. Saves to disk.
4. Returns the state.

---

### `prepare_simulation(simulation_id, simulation_requirement, document_text, ...) → SimulationState`

**Main pipeline method** — the backbone of the prepare API endpoint. Runs in a background thread.

| Parameter | Type | Description |
|-----------|------|-------------|
| `simulation_id` | `str` | Target simulation |
| `simulation_requirement` | `str` | User's natural-language goal for the simulation |
| `document_text` | `str` | Original source document text |
| `defined_entity_types` | `Optional[List[str]]` | Pre-set entity types to filter by |
| `use_llm_for_profiles` | `bool` | Whether to enrich profiles with LLM |
| `progress_callback` | `Optional[callable]` | Called with `(stage, pct, msg, ...)` for SSE streaming |
| `parallel_profile_count` | `int` | Concurrent profile generation threads (default `3`) |

**Stage 1 — Entity Reading:**
- Instantiates `ZepEntityReader`.
- Calls `filter_defined_entities(graph_id, defined_entity_types, enrich_with_edges=True)`.
- Updates `state.entities_count`, `state.entity_types`.
- If zero entities found → sets status `FAILED` and returns early.

**Stage 2 — Profile Generation:**
- Determines real-time output format: Reddit JSON (preferred) or Twitter CSV.
- Calls `OasisProfileGenerator.generate_profiles_from_entities()` with `progress_callback`, `parallel_count`, and `realtime_output_path`.
- Saves final profiles to `reddit_profiles.json` and/or `twitter_profiles.csv` in the simulation directory.

**Stage 3 — Config Generation:**
- Calls `SimulationConfigGenerator.generate_config()` with entities, requirement, and document text.
- Writes `simulation_config.json` to the simulation directory.
- Stores `config_reasoning` in state.

**Completion:**
- Sets `state.status = READY`.
- Saves final state to disk.

**On any exception:**
- Sets `state.status = FAILED`.
- Saves error message.
- Re-raises so the background thread caller can log it.

---

### `get_simulation(simulation_id) → Optional[SimulationState]`

Thin wrapper over `_load_simulation_state()`.

---

### `list_simulations(project_id=None) → List[SimulationState]`

Scans `SIMULATION_DATA_DIR/` for subdirectories, loads each `state.json`, and filters by `project_id` if given. Skips hidden files (`.DS_Store` etc.) and non-directory entries.

---

### `get_profiles(simulation_id, platform="reddit") → List[Dict]`

Reads `{platform}_profiles.json` from the simulation directory: returns the parsed JSON array or empty list if not yet generated.

---

### `get_simulation_config(simulation_id) → Optional[Dict]`

Reads `simulation_config.json` from the simulation directory. Returns `None` if not yet generated.

---

### `get_run_instructions(simulation_id) → Dict`

Returns a dictionary of shell commands and human-readable instructions for running the simulation manually. Commands point to scripts in `backend/scripts/`:
- `run_twitter_simulation.py`
- `run_reddit_simulation.py`
- `run_parallel_simulation.py`

---

## Filesystem Layout (per simulation)

```
uploads/simulations/<sim_id>/
    state.json                  ← SimulationState, updated throughout lifecycle
    reddit_profiles.json        ← List of OasisAgentProfile (JSON)
    twitter_profiles.csv        ← Twitter-format profile CSV (OASIS requirement)
    simulation_config.json      ← Full SimulationParameters JSON
```

---

## Related Files

- [`api/simulation.py`](../api/simulation.md) — HTTP routes that call `SimulationManager`
- [`services/simulation_runner.py`](simulation_runner.md) — Launches the OASIS subprocess after preparation
- [`services/zep_entity_reader.py`](zep_entity_reader.md) — Stage 1 entity reading
- [`services/oasis_profile_generator.py`](oasis_profile_generator.md) — Stage 2 profile generation
- [`services/simulation_config_generator.py`](simulation_config_generator.md) — Stage 3 config generation
