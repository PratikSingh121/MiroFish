# `backend/app/services/simulation_config_generator.py` — Simulation Config Generator

## Overview

`simulation_config_generator.py` uses an LLM to **automatically generate all simulation parameters** — eliminating the need for users to manually tune anything. Given the simulation entities and the simulation requirement, it produces a complete `SimulationParameters` object covering:

- **Time simulation** — how many rounds, how many simulated hours, activity patterns by time of day (modelled on Chinese daily rhythms).
- **Per-agent configuration** — activity level, posting frequency, active hours, sentiment bias, stance toward the scenario, influence weight.
- **Event configuration** — initial seed posts, scheduled events, hot-topic keywords.
- **Platform configuration** — recommendation algorithm weights, viral spreading thresholds, echo-chamber strength.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `OpenAI` | `openai` SDK | LLM calls |
| `EntityNode`, `ZepEntityReader` | [`services/zep_entity_reader.py`](zep_entity_reader.md) | Entity context |
| `Config` | [`config.py`](../config.md) | API keys and model |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## Constants

### `CHINA_TIMEZONE_CONFIG`

A dictionary modelling typical Chinese social-media usage patterns by hour of day:

| Time Slot | Hours | Activity Multiplier |
|-----------|-------|---------------------|
| Dead (night) | 0–5 | 0.05 — almost no one |
| Morning | 6–8 | 0.4 — gradual ramp-up |
| Work | 9–18 | 0.7 — moderate activity |
| Peak (evening) | 19–22 | 1.5 — most active |
| Night | 23 | 0.5 — winding down |

This is applied when generating per-agent `active_hours` and when the config generator sets `agents_per_hour_min/max`.

---

## Data Structures

### `AgentActivityConfig`

Per-agent simulation parameters:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `agent_id` | `int` | — | Sequential agent ID |
| `entity_uuid` | `str` | — | Corresponding Zep entity UUID |
| `entity_name` | `str` | — | Entity display name |
| `entity_type` | `str` | — | Ontology type |
| `activity_level` | `float` | `0.5` | 0–1 overall activity intensity |
| `posts_per_hour` | `float` | `1.0` | Expected post rate |
| `comments_per_hour` | `float` | `2.0` | Expected comment rate |
| `active_hours` | `List[int]` | `8-22` | Hours when agent is active |
| `response_delay_min/max` | `int` | `5/60` | How quickly agent reacts (sim minutes) |
| `sentiment_bias` | `float` | `0.0` | -1 to 1, negative to positive posting tendency |
| `stance` | `str` | `"neutral"` | `supportive`, `opposing`, `neutral`, `observer` |
| `influence_weight` | `float` | `1.0` | How visible the agent's posts are |

### `TimeSimulationConfig`

| Field | Default | Description |
|-------|---------|-------------|
| `total_simulation_hours` | `72` | Total simulated time span (3 days) |
| `minutes_per_round` | `60` | Each round represents 60 real minutes |
| `agents_per_hour_min/max` | `5/20` | Number of agents activating each round |
| `peak_hours` | `[19,20,21,22]` | Evening peak |
| `off_peak_hours` | `[0,1,2,3,4,5]` | Dead of night |
| `peak_activity_multiplier` | `1.5` | Activity boost during peak |
| `off_peak_activity_multiplier` | `0.05` | Near-zero during dead hours |

### `EventConfig`

| Field | Description |
|-------|-------------|
| `initial_posts` | Seed posts that appear at round 0 to kick off the narrative |
| `scheduled_events` | Posts/events injected at specific simulation hours |
| `hot_topics` | Keywords that agents will naturally search for |
| `narrative_direction` | High-level description of how the scenario should unfold |

### `PlatformConfig`

Per-platform recommendation algorithm settings:

| Field | Default | Description |
|-------|---------|-------------|
| `recency_weight` | `0.4` | How much newer posts are boosted |
| `popularity_weight` | `0.3` | How much highly-liked posts are boosted |
| `relevance_weight` | `0.3` | How much on-topic posts are boosted |
| `viral_threshold` | `10` | Interactions before virality triggers |
| `echo_chamber_strength` | `0.5` | How much agents prefer similar opinion posts |

### `SimulationParameters`

The top-level container aggregating all of the above:

```python
@dataclass
class SimulationParameters:
    simulation_id: str
    project_id: str
    graph_id: str
    simulation_requirement: str
    time_config: TimeSimulationConfig
    agent_configs: List[AgentActivityConfig]
    event_config: EventConfig
    twitter_config: Optional[PlatformConfig]
    reddit_config: Optional[PlatformConfig]
    llm_model: str
    llm_base_url: str
    generated_at: str
    generation_reasoning: str   # LLM's explanation of parameter choices
```

#### `SimulationParameters.to_dict()` / `to_json()`

Converts the entire nested structure to a dict or JSON string for saving to `simulation_config.json`.

---

## `SimulationConfigGenerator` Class

### `generate(entities, simulation_id, project_id, graph_id, simulation_requirement, document_text, ...) → SimulationParameters`

**Main public method.** Uses a **step-by-step generation strategy** to avoid hitting LLM context limits:

| Step | LLM Call | Generates |
|------|----------|-----------|
| 1 | `_generate_time_config()` | `TimeSimulationConfig` |
| 2 | `_generate_event_config()` | `EventConfig` |
| 3 | `_generate_agent_configs_batch()` | `AgentActivityConfig` per entity (batched) |
| 4 | `_generate_platform_configs()` | `PlatformConfig` for Twitter and Reddit |

Each step creates a focused prompt with only the information needed for that aspect of the config, reducing hallucination and token usage.

The `generation_reasoning` field is populated with the LLM's commented explanation of the parameter choices, which is useful for debugging and understanding why the simulation was configured the way it was.

---

## Related Files

- [`services/simulation_manager.py`](simulation_manager.md) — calls `SimulationConfigGenerator.generate()` during prepare
- [`services/zep_entity_reader.py`](zep_entity_reader.md) — provides `EntityNode` objects
- [`api/simulation.py`](../api/simulation.md) — exposes `/config` and `/config/realtime` endpoints
