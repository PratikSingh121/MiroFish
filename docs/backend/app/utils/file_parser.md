# `backend/app/utils/file_parser.py` — File Parser

## Overview

`file_parser.py` is the backend’s document-ingestion utility. It extracts raw text from uploaded files and splits that text into graph-friendly chunks.

Supported formats:
- PDF
- Markdown
- plain text

The file is careful about encoding issues, which matters because uploaded text files may not be UTF-8.

---

## Top-Level Helper

### `_read_text_with_fallback(file_path) -> str`

Robust text decoding pipeline.

Fallback order:
1. try UTF-8 directly
2. try `charset_normalizer`
3. try `chardet`
4. fall back to UTF-8 with `errors='replace'`

This function is what makes `.md` and `.txt` ingestion resilient to mixed encodings.

---

## `FileParser`

### `SUPPORTED_EXTENSIONS`

The parser accepts `.pdf`, `.md`, `.markdown`, and `.txt`.

### `extract_text(file_path) -> str`

Main entry point.

Behavior:
1. validates file existence
2. checks extension support
3. dispatches to the correct extractor based on suffix

### `_extract_from_pdf(file_path) -> str`

Uses `fitz` from PyMuPDF to iterate through pages and collect extracted text.

Raises an `ImportError` with an installation hint if PyMuPDF is missing.

### `_extract_from_md(file_path) -> str`

Reads markdown as plain text using `_read_text_with_fallback()`.

### `_extract_from_txt(file_path) -> str`

Reads plain text using the same fallback decoder.

### `extract_from_multiple(file_paths) -> str`

Loops through many files, extracts each one, and joins them into one string with document headers like:

```text
=== 文档 1: filename ===
```

If a file fails to parse, the returned output contains a failure marker instead of crashing the entire batch.

---

## Chunking Helper

### `split_text_into_chunks(text, chunk_size=500, overlap=50) -> List[str]`

Splits a long string into overlapping chunks.

Important behavior:
- tries to break near sentence boundaries when possible
- supports both Chinese and English punctuation
- preserves overlap between chunks to reduce context loss
- returns an empty list for blank input

This is the function used before uploading document episodes to Zep.

---

## Related Files

- [`services/text_processor.py`](../services/text_processor.md) wraps this module for service-layer use
- [`services/graph_builder.py`](../services/graph_builder.md) consumes chunked text for graph construction
- [`api/graph.py`](../api/graph.md) triggers parsing through upload workflows
