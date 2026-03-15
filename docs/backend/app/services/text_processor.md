# `backend/app/services/text_processor.py` — Text Processor

## Overview

`text_processor.py` is a thin facade over the lower-level parsing helpers in [`utils/file_parser.py`](../utils/file_parser.md). It gives the rest of the backend one small service with three responsibilities:

- extract text from uploaded files
- split long text into chunks for graph ingestion
- normalize and summarize raw text

Because the implementation is intentionally small, this file mainly exists to keep higher-level services decoupled from the parsing utility module.

---

## `TextProcessor`

### `extract_from_files(file_paths) -> str`

Calls `FileParser.extract_from_multiple()` and returns one merged text blob containing every uploaded document.

### `split_text(text, chunk_size=500, overlap=50) -> List[str]`

Delegates to `split_text_into_chunks()`.

Purpose:
- create graph-ingestion chunks
- preserve some overlap so context is not lost between adjacent chunks

### `preprocess_text(text) -> str`

Normalizes text before LLM or graph ingestion.

Behavior:
1. converts Windows and old-Mac line endings to `\n`
2. collapses runs of 3+ blank lines to 2
3. strips leading/trailing whitespace on each line
4. trims the final result

### `get_text_stats(text) -> dict`

Returns lightweight summary metrics:
- `total_chars`
- `total_lines`
- `total_words`

---

## Related Files

- [`utils/file_parser.py`](../utils/file_parser.md) does the actual file decoding and chunking
- [`services/graph_builder.py`](graph_builder.md) uses chunked text when sending episodes to Zep
- [`api/graph.py`](../api/graph.md) starts the upload and graph-build workflow
