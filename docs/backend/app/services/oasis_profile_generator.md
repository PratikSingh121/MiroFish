# `backend/app/services/oasis_profile_generator.py` — OASIS Agent Profile Generator

## Overview

`oasis_profile_generator.py` converts the typed `EntityNode` objects from the Zep graph into **OASIS Agent Profiles** — richly-detailed persona descriptions that tell each simulated agent who they are, how they think, what they care about, and how they behave on social media.

The quality of these personas directly determines the realism and predictive value of the simulation. A poorly-described agent will post generic content; a well-described one will post content that reflects their real-world role, relationships, and motivations.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `OpenAI` | `openai` SDK | Direct LLM calls for profile generation |
| `Zep` | `zep_cloud` SDK | Search Zep graph for richer context |
| `EntityNode`, `ZepEntityReader` | [`services/zep_entity_reader.py`](zep_entity_reader.md) | Source entity data |
| `Config` | [`config.py`](../config.md) | API keys and model config |
| `get_logger` | [`utils/logger.py`](../utils/logger.md) | Logging |

---

## `OasisAgentProfile` Dataclass

The complete persona for a single agent:

```python
@dataclass
class OasisAgentProfile:
    # Identity
    user_id: int
    user_name: str      # @handle (ASCII, platform-friendly)
    name: str           # Display name
    bio: str            # Short bio (Twitter-style)
    persona: str        # Detailed personality description for LLM prompt

    # Reddit stats (influence social media behaviour)
    karma: int = 1000

    # Twitter stats
    friend_count: int = 100
    follower_count: int = 150
    statuses_count: int = 500

    # Demographics
    age: Optional[int] = None
    gender: Optional[str] = None
    mbti: Optional[str] = None
    country: Optional[str] = None
    profession: Optional[str] = None
    interested_topics: List[str] = []

    # Provenance
    source_entity_uuid: Optional[str] = None
    source_entity_type: Optional[str] = None
    created_at: str = "YYYY-MM-DD"
```

### `to_reddit_format() → Dict`

Returns a dict with keys expected by the OASIS Reddit platform: `user_id`, `username`, `name`, `bio`, `persona`, `karma`, `created_at`, plus optional demographic fields.

### `to_twitter_format() → Dict`

Same but with `friend_count`, `follower_count`, `statuses_count` instead of `karma`.

### `to_dict() → Dict`

Returns the complete internal representation.

---

## `OasisProfileGenerator` Class

### Class-Level Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MBTI_TYPES` | 16 types | Randomly assigned if LLM doesn't specify |
| `COUNTRIES` | 11 countries | Randomly assigned if LLM doesn't specify |
| `INDIVIDUAL_ENTITY_TYPES` | `[student, person, professor, ...]` | Entity types that represent single people |
| `GROUP_ENTITY_TYPES` | `[university, company, organization, ...]` | Entity types representing groups/orgs |

### `__init__(...)`

Initialises both an `OpenAI` client (for profile generation LLM calls) and a `Zep` client (for context retrieval). Accepts optional overrides for `api_key`, `base_url`, `model_name`, `zep_api_key`, `graph_id`.

---

### `generate_profiles(entities, simulation_requirement, document_text, use_llm=True, parallel_count=5, progress_callback=None) → List[OasisAgentProfile]`

**Top-level method.** Generates profiles for all entities.

**Process:**
1. Assigns sequential `user_id` values (starting from 1).
2. If `use_llm=True`, processes entities in batches of `parallel_count` using **concurrent threads** (via `concurrent.futures.ThreadPoolExecutor`) to speed up profile generation.
3. For each entity calls `_generate_single_profile()`.
4. Falls back to `_generate_basic_profile()` if any LLM call fails.
5. Calls `progress_callback(done_count, total_count)` after each batch.

---

### `_generate_single_profile(entity, user_id, simulation_requirement, document_text) → OasisAgentProfile`

Generates a rich profile for one entity:

1. **Retrieves Zep context** — calls `ZepEntityReader.get_entity_with_context()` to get the entity's related edges and neighbours for richer background.
2. **Determines entity category** — checks if the entity type is in `INDIVIDUAL_ENTITY_TYPES` or `GROUP_ENTITY_TYPES` to tailor the prompt.
3. **Calls LLM** — sends a detailed prompt containing:
   - The entity's name, type, Zep summary, and attributes.
   - Related entities and relationships.
   - The simulation requirement (so the persona aligns with the scenario).
   - Instructions to generate: bio, persona, age, gender, MBTI, profession, interested_topics.
4. **Parses JSON response** — extracts all fields and builds the `OasisAgentProfile`.
5. **Generates platform-specific fields** — normalises `user_name` to a URL-safe ASCII handle, assigns realistic `karma`/follower counts based on entity prominence.

---

### `_generate_basic_profile(entity, user_id) → OasisAgentProfile`

**Fallback method** when the LLM call fails. Creates a minimal profile using only the entity's name and type from the Zep node data. Randomly assigns MBTI and country from the class constants.

---

### `save_reddit_profiles(profiles, output_path)`

Serializes profiles to `reddit_profiles.json` using `profile.to_reddit_format()` for each profile.

### `save_twitter_profiles(profiles, output_path)`

Serializes profiles to `twitter_profiles.csv` in the format OASIS expects.

---

## Design Notes

### Individual vs Group Entities

The generator distinguishes between:
- **Individual entities** (e.g. `Student`, `Person`) — prompts LLM to create a specific human persona with personal history, emotions, opinions.
- **Group entities** (e.g. `University`, `Company`) — prompts LLM to create an "official spokesperson" or "PR account" persona that speaks for the institution.

### Prompt Engineering

The profile prompt is one of the most important prompts in MiroFish. It explicitly asks for:
- Detailed backstory relevant to the simulation scenario.
- Emotional disposition toward the events described.
- Social media behavior patterns (when they post, what tone they use, how they react to others).
- Specific `interested_topics` that the agent will search for and engage with.

---

## Related Files

- [`services/zep_entity_reader.py`](zep_entity_reader.md) — provides `EntityNode` objects and Zep context retrieval
- [`services/simulation_manager.py`](simulation_manager.md) — calls `generate_profiles()` during simulation prepare
- [`api/simulation.py`](../api/simulation.md) — coordinates the prepare workflow
