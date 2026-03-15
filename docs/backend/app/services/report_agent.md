# `backend/app/services/report_agent.py` — Report Agent

## Overview

`report_agent.py` is the report-generation engine for MiroFish. It turns a finished simulation into a structured markdown prediction report and supports follow-up chat over that report.

The file combines four concerns:
- structured logging of the report-generation workflow
- console-log capture for UI playback
- a ReACT-style `ReportAgent` that plans and writes the report
- a `ReportManager` that persists outlines, progress, sections, logs, and the final assembled markdown

This is one of the core files in the backend because it connects the graph retrieval layer, the LLM layer, and the frontend polling UI.

---

## Logging Classes

### `ReportLogger`

Writes structured JSONL events to `agent_log.jsonl` inside the report folder.

#### Purpose
Each line is one machine-readable event so the frontend can reconstruct what the agent is doing in near real time.

#### Core helpers
- `__init__(report_id)` sets the target file path
- `_ensure_log_file()` creates the directory
- `_get_elapsed_time()` returns runtime since logger creation
- `log(action, stage, details, section_title=None, section_index=None)` writes one JSONL record

#### Specialized event methods
These all delegate to `log()` with a stable schema:
- `log_start()`
- `log_planning_start()`
- `log_planning_context()`
- `log_planning_complete()`
- `log_section_start()`
- `log_react_thought()`
- `log_tool_call()`
- `log_tool_result()`
- `log_llm_response()`
- `log_section_content()`
- `log_section_full_complete()`
- `log_report_complete()`
- `log_error()`

### `ReportConsoleLogger`

Adds a file handler so normal Python logger output is also copied into `console_log.txt` for the same report.

#### Methods
- `__init__(report_id)` prepares the output file and attaches the handler
- `_ensure_log_file()` creates the folder
- `_setup_file_handler()` adds the handler to `mirofish.report_agent` and `mirofish.zep_tools`
- `close()` removes and closes the handler
- `__del__()` is a final cleanup fallback

This exists because the frontend shows both structured agent events and raw console-style logs.

---

## Report Data Structures

### `ReportStatus`

Lifecycle values:
- `PENDING`
- `PLANNING`
- `GENERATING`
- `COMPLETED`
- `FAILED`

### `ReportSection`

Represents one section in the final report.

Methods:
- `to_dict()`
- `to_markdown(level=2)`

### `ReportOutline`

Represents the full planned structure.

Fields:
- `title`
- `summary`
- `sections`

Methods:
- `to_dict()`
- `to_markdown()`

### `Report`

Top-level report record.

Fields include:
- `report_id`
- `simulation_id`
- `graph_id`
- `simulation_requirement`
- `status`
- optional `outline`
- `markdown_content`
- `created_at`
- `completed_at`
- optional `error`

Method:
- `to_dict()`

---

## Prompt Constants

This file stores a large number of prompt templates and tool descriptions used by the report workflow.

Their role is to enforce:
- a future-prediction framing for the report
- strict formatting rules
- mandatory tool usage before writing content
- report planning and section-writing behavior
- chat behavior after the report is finished

The important design point is that prompt engineering is kept in one file, close to the orchestration code that consumes it.

---

## `ReportAgent`

### Role

`ReportAgent` is a ReACT-style writer with access to graph retrieval tools. It plans a report outline first, then generates sections one by one, calling retrieval tools before committing final prose.

### Class settings
- `MAX_TOOL_CALLS_PER_SECTION = 5`
- `MAX_REFLECTION_ROUNDS = 3`
- `MAX_TOOL_CALLS_PER_CHAT = 2`

### `__init__(graph_id, simulation_id, simulation_requirement, llm_client=None, zep_tools=None)`

Stores report context, initializes or injects:
- `LLMClient`
- `ZepToolsService`
- tool metadata from `_define_tools()`

Also prepares placeholders for `ReportLogger` and `ReportConsoleLogger`.

### `_define_tools() -> Dict[str, Dict[str, Any]]`

Returns the tool registry used in prompts and runtime dispatch.

Supported tools:
- `insight_forge`
- `panorama_search`
- `quick_search`
- `interview_agents`

### `_execute_tool(tool_name, parameters, report_context='') -> str`

Runtime dispatch for agent tool calls.

Behavior:
- calls the corresponding method on `ZepToolsService`
- normalizes parameter types where needed
- returns tool results as text, because the LLM consumes text observations
- supports a few older tool names for backward compatibility by redirecting them to newer implementations

### `VALID_TOOL_NAMES`

Set used by the parser to validate recovered tool-call JSON.

### `_parse_tool_calls(response) -> List[Dict[str, Any]]`

Extracts tool calls from LLM output.

Supported formats:
- `<tool_call>{...}</tool_call>`
- raw JSON fallback if the model skipped XML tags

### `_is_valid_tool_call(data) -> bool`

Normalizes accepted key names such as `tool` and `params` into the canonical `name` and `parameters` shape.

### `_get_tools_description() -> str`

Builds a textual description of all tools for prompt injection.

### `plan_outline(progress_callback=None) -> ReportOutline`

Planning stage.

Behavior:
1. gets simulation context via `zep_tools.get_simulation_context()`
2. fills the planning prompt
3. calls `llm.chat_json()`
4. converts the JSON into `ReportOutline`
5. falls back to a default 3-section outline if planning fails

This is the first phase of report generation.

### `_generate_section_react(section, outline, previous_sections, progress_callback=None, section_index=0) -> str`

Core section-writing loop.

This is the most important method in the file.

#### ReACT workflow
1. build system and user prompts for the current section
2. call the LLM
3. parse tool calls if present
4. execute at most one tool per iteration
5. append an observation back into the message history
6. continue until enough tools have been used and the model emits a final answer
7. if the loop stalls, force a final answer at the end

#### Important safeguards
- rejects mixed responses that contain both a tool call and a final answer
- enforces a minimum number of tool calls before accepting a final answer
- enforces a maximum tool-call budget
- accepts plain text as a final answer when the model forgets the `Final Answer:` prefix
- records every major step into `ReportLogger`

### `generate_report(progress_callback=None, report_id=None) -> Report`

Top-level end-to-end report generation workflow.

Steps:
1. create or accept a report ID
2. initialize report metadata and report folder
3. create `ReportLogger` and `ReportConsoleLogger`
4. plan the outline
5. save `outline.json`
6. generate each section with `_generate_section_react()`
7. save each section as soon as it completes
8. assemble the final markdown with `ReportManager.assemble_full_report()`
9. update report status and progress files throughout
10. return a completed or failed `Report` object

A key design decision here is streaming persistence: each section is saved as soon as it exists, so the frontend can show incremental progress instead of waiting for the full report.

### `chat(message, chat_history=None) -> Dict[str, Any]`

Post-report conversational mode.

Behavior:
- loads the generated report if available
- injects report content into the chat system prompt
- allows a smaller number of retrieval tool calls than section generation
- returns a response plus tool-call/source metadata

This is what powers the interactive report-agent chat in step 5.

---

## `ReportManager`

### Role

`ReportManager` is the persistence and file-layout layer for reports.

### Filesystem layout

```text
uploads/reports/<report_id>/
  meta.json
  outline.json
  progress.json
  section_01.md
  section_02.md
  ...
  full_report.md
  agent_log.jsonl
  console_log.txt
```

### Path helpers

These methods only compute paths or ensure directories exist:
- `_ensure_reports_dir()`
- `_get_report_folder(report_id)`
- `_ensure_report_folder(report_id)`
- `_get_report_path(report_id)`
- `_get_report_markdown_path(report_id)`
- `_get_outline_path(report_id)`
- `_get_progress_path(report_id)`
- `_get_section_path(report_id, section_index)`
- `_get_agent_log_path(report_id)`
- `_get_console_log_path(report_id)`

### Log readers
- `get_console_log(report_id, from_line=0)` returns incremental raw log lines
- `get_console_log_stream(report_id)` returns all console lines
- `get_agent_log(report_id, from_line=0)` returns incremental JSONL records
- `get_agent_log_stream(report_id)` returns all agent log entries

These are polled by the frontend during step 4.

### Content writers
- `save_outline(report_id, outline)` writes `outline.json`
- `save_section(report_id, section_index, section)` writes one section markdown file
- `_clean_section_content(content, section_title)` removes duplicated headings and demotes nested markdown headings into bold text
- `update_progress(report_id, status, progress, message, current_section=None, completed_sections=None)` writes `progress.json`

### Content readers and assembly
- `get_progress(report_id)` reads `progress.json`
- `get_generated_sections(report_id)` reads every saved `section_XX.md`
- `assemble_full_report(report_id, outline)` builds `full_report.md` from outline plus saved sections
- `_post_process_report(content, outline)` performs final heading cleanup and spacing normalization

### Report lifecycle helpers
- `save_report(report)` writes `meta.json`, optional outline, and optional full markdown
- `get_report(report_id)` reconstructs a `Report` object from disk
- `get_report_by_simulation(simulation_id)` finds the report associated with one simulation
- `list_reports(simulation_id=None, limit=50)` returns reports sorted by creation time
- `delete_report(report_id)` removes a report folder, with backward compatibility for the older flat-file format

---

## Related Files

- [`services/zep_tools.py`](zep_tools.md) supplies the retrieval tools used by `ReportAgent`
- [`utils/llm_client.py`](../utils/llm_client.md) supplies all model calls
- [`api/report.py`](../api/report.md) exposes report-generation, polling, download, and chat endpoints
- [`frontend/src/components/Step4Report.vue`](../../../frontend/src/components/Step4Report.md) visualizes the incremental logs and sections
- [`frontend/src/components/Step5Interaction.vue`](../../../frontend/src/components/Step5Interaction.md) uses the post-report chat flow
