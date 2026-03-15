# `backend/app/services/ontology_generator.py` — Ontology Generation Service

## Overview

`ontology_generator.py` uses an LLM to analyse the uploaded documents and the user's simulation requirement, then produces a **structured ontology** — a schema of entity types and relationship types that will guide how the knowledge graph is constructed.

The ontology is critical to the whole pipeline: it determines which entities Zep will recognise in the text, which relationship types it will extract, and which entities will later become simulation agents.

---

## Key Concept: What is an Ontology Here?

In MiroFish's context the "ontology" is a typed schema of:

- **Entity types** — categories of real-world actors (people, organisations, institutions) who can post on social media. Each type has a name, description, attributes, and examples. **Exactly 10 types** are required: 8 domain-specific and 2 catch-all types (`Person` and `Organization`).
- **Edge types** — categories of relationships between entities (e.g. `WORKS_FOR`, `REPORTS_ON`). 6–10 types are generated.

The LLM is carefully prompted to only define entities that can **speak, post, and interact** on social media — not abstract concepts.

---

## System Prompt (`ONTOLOGY_SYSTEM_PROMPT`)

A very detailed prompt (~200 lines) that:
1. Explains the social-media simulation context.
2. Lists valid entity categories (individuals, organisations) and invalid ones (concepts, topics).
3. Specifies the **exact JSON output format**.
4. Enforces rules:
   - Exactly 10 entity types with `Person` and `Organization` as the last two.
   - 6–10 edge types.
   - 1–3 attributes per entity type.
   - Attribute names must not use Zep reserved words (`name`, `uuid`, `group_id`, `created_at`, `summary`).
5. Provides reference entity/relation type lists as inspiration.

---

## Dependencies

| Import | Source | Purpose |
|--------|--------|---------|
| `LLMClient` | [`utils/llm_client.py`](../utils/llm_client.md) | OpenAI-compatible LLM calls |

---

## `OntologyGenerator` Class

### `__init__(llm_client=None)`

Accepts an optional `LLMClient` instance. If none is provided, creates a default client from `Config`.

### `generate(document_texts, simulation_requirement, additional_context=None) → Dict`

**Main public method.** Produces the full ontology dictionary.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `document_texts` | `List[str]` | List of text content from the uploaded documents |
| `simulation_requirement` | `str` | User's description of what they want to simulate/predict |
| `additional_context` | `Optional[str]` | Any extra background information |

**Process:**

1. Calls `_build_user_message()` to assemble the user turn of the LLM conversation.
2. Calls `LLMClient.chat_json()` with the system prompt + user message at `temperature=0.3`. Low temperature ensures deterministic, structured output.
3. Returns the parsed JSON dict which should contain `entity_types`, `edge_types`, and `analysis_summary`.

**Returns:**
```json
{
  "entity_types": [
    {
      "name": "Student",
      "description": "University student posting on social media",
      "attributes": [{"name": "university", "type": "text", "description": "..."}],
      "examples": ["Zhang Wei", "Li Ming"]
    },
    ...
  ],
  "edge_types": [
    {
      "name": "STUDIES_AT",
      "description": "Student enrolled at a university",
      "source_targets": [{"source": "Student", "target": "University"}],
      "attributes": []
    },
    ...
  ],
  "analysis_summary": "This document describes..."
}
```

### `_build_user_message(document_texts, simulation_requirement, additional_context) → str`

Private helper. Constructs the user message sent to the LLM by concatenating:
- The simulation requirement.
- All document texts (truncated if necessary to fit context window).
- The additional context (if any).

---

## Design Decisions

- **Synchronous call:** Unlike graph building, ontology generation is *synchronous* (no background thread). The HTTP endpoint for ontology generation waits for the LLM response before returning. This is acceptable because ontology generation is fast (a single LLM call, no Zep API calls).
- **Temperature 0.3:** Low temperature reduces creativity and ensures the JSON is well-formed.
- **JSON mode:** Uses `LLMClient.chat_json()` which requests `response_format={"type": "json_object"}` and handles post-processing to strip markdown fences.

---

## Related Files

- [`api/graph.py`](../api/graph.md) — calls `OntologyGenerator.generate()` in `generate_ontology()`
- [`utils/llm_client.py`](../utils/llm_client.md) — LLM wrapper used for the analysis
- [`services/graph_builder.py`](graph_builder.md) — consumes the ontology to configure Zep's schema
