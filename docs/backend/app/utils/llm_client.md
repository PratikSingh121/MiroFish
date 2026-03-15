# `backend/app/utils/llm_client.py` — LLM Client

## Overview

`llm_client.py` standardizes all LLM access behind one small wrapper around the OpenAI-compatible SDK. The rest of the backend can call one client regardless of whether the actual model comes from OpenAI or an OpenAI-style compatible endpoint.

The file also contains defensive cleanup for malformed model output, especially JSON wrapped in markdown fences or `<think>` tags.

---

## `LLMClient`

### `__init__(api_key=None, base_url=None, model=None)`

Resolves configuration from explicit arguments first, then falls back to [`app/config.py`](../config.md):
- `LLM_API_KEY`
- `LLM_BASE_URL`
- `LLM_MODEL_NAME`

Raises `ValueError` if no API key is available.

Creates an `OpenAI` client instance configured for the selected base URL.

### `chat(messages, temperature=0.7, max_tokens=4096, response_format=None) -> str`

Generic chat-completions wrapper.

Behavior:
1. builds the OpenAI request payload
2. optionally attaches `response_format`
3. returns the first completion’s text
4. strips `<think>...</think>` blocks from some reasoning models before returning

This is the main method used by ontology generation, profile generation, config generation, and the report agent.

### `chat_json(messages, temperature=0.3, max_tokens=4096) -> Dict[str, Any]`

JSON-specialized wrapper.

Behavior:
1. calls `chat()` with `response_format={"type": "json_object"}`
2. removes optional markdown code fences like ```json
3. parses the cleaned response with `json.loads`
4. raises `ValueError` if the returned string is still not valid JSON

This method is used wherever the backend expects structured model output instead of prose.

---

## Related Files

- [`services/ontology_generator.py`](../services/ontology_generator.md)
- [`services/oasis_profile_generator.py`](../services/oasis_profile_generator.md)
- [`services/simulation_config_generator.py`](../services/simulation_config_generator.md)
- [`services/report_agent.py`](../services/report_agent.md)
