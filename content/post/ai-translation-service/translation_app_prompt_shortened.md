You are a senior full-stack coding agent. Build a internet accessible HTTPS translation service on mainstream Linux OS that translates Simplified Chinese Markdown → English using ModelArk's LLM service. The service must support BOTH a browser UI and an HTTP API for scripts/services.

============================================================
0) Environment / Non-functional
============================================================
- OS: Linux, Python 3 available.
- Network: 
  - Service can be accessed via internet. 
  - Use ASGI (uvicorn) for concurrency.
- Secrets: User need to provide their own LLM secrets, base url and other info, for example
  - API_KEY (required)
  - BASE_URL (required)
  - MODEL_ID (default)
  - TIMEOUT_SECONDS (default 180)
- Keep dependencies minimal: for example, FastAPI + Uvicorn + simple HTML (Jinja2/static) + minimal JS.
- Storage must be lightweight. You MUST implement a server-side “source version repository” for Chinese files (filesystem or SQLite). Also design the storage module so it can be extended later to store translation memory (zh→en mapping), but DO NOT implement TM now.

============================================================
1) Required Features
============================================================

A) Full translation
- Input: upload a Chinese .md file OR paste Chinese Markdown text.
- Output: English Markdown file (downloadable).
- MUST preserve Markdown structure and MUST NOT translate/alter:
  - fenced code blocks ```...``` (including language tag like ```python)
  - inline code `...`
  - URLs and link targets: [text](URL) where URL stays identical
  - file paths
- Headings/lists/tables/links/images must keep their formatting; only natural-language text is translated.

B) Incremental translation
- The server maintains a repository of Chinese source files keyed by a user-provided repository_path.
- Full translation mode: after translation, store the input Chinese file into the repo at repository_path.
- Incremental mode:
  1) Load previous Chinese file from repo by repository_path.
  2) Compare previous vs current Chinese file (line-by-line diff is OK; normalize trailing spaces if desired).
  3) Translate ONLY the changed lines (no TM reuse required).
    - For code block changes, find them out but do not translate them, return the updated code blocks in the output.
  4) Output ONLY the changed lines as zh/en pairs (e.g., JSON array [{zh, en}, ...] or a small table; choose one and document it).
  5) Replace the stored repo file with the new Chinese file.
- If no previous version exists for repository_path, return an error telling the user/client to run full translation first.

C) Markdown Preprocessing
- Before sending the content for translation, replace all the code blocks and html tags with placeholder, such as <attribute id=xxx>; You design an effective way
- Ensure the placeholders will not affect model's translation
- Ensure original code blocks, html tags can be safely add back to the translated content.

C) Chunked translation
- Preprocess and translate in chunks to fit context limits.
- User specifies the chunk number, from 1-20; this number indicates how many chunks the contente will be divided
- For each translation tasks, send one chunk to the model to translate in one request; e.g., 8 chunks will send 8 requests to model.
- Translate chunks sequentially and concatenate in order.

D) Glossary — FULL LIST TO MODEL EACH REQUEST
- Optional glossary CSV: source_term,target_term (Chinese → required English).
- For EVERY translation request (full or incremental, each chunk or each changed-line call), pass the ENTIRE glossary list into the system prompt.
- Instruct the model: “Translate these terms exactly as specified; do not deviate.”
- Do NOT pre-replace terms in the source text.
- Optionally add a post-check to detect glossary violations and warn/error, but do not auto-rewrite terms.

============================================================
2) ModelArk Integration
============================================================
- Implement a ModelArk client module that:
  - Builds requests to the ModelArk Responses API using env config. 
  - Use the OpenAI compatible API (Python SDK) so later can configure to use models provided by other providers.
  - Uses a strong system prompt enforcing: zh→en, preserve Markdown, skip code/inline/URLs/paths, and obey glossary exactly.
  - Implements retries (exponential backoff) for transient errors (429/5xx) and clear error messages.
  - Supports chunking (send chunks in order; combine outputs).

============================================================
3) Interfaces
============================================================

3.1 Web UI (GET /)
Single page with:
- markdown input: file upload (.md) + textarea (either one).
- repository_path input (required; used for saving/loading the Chinese file in repo).
- Checkbox “Incremental mode”.
- Glossary CSV upload (optional).
- Translate button + loading indicator.
- Output:
  - Full mode: download translated English .md (and optional preview).
  - Incremental mode: show changed-line zh/en pairs and allow download (json/csv/md).
- when refreshing, translation ongoing tasks should remain

3.2 API
- POST /api/translate (multipart/form-data preferred; JSON allowed for text-only clients)
  Inputs:
    - updated_file OR updated_text
    - repository_path (required)
    - incremental (bool, default false)
    - glossary_csv (optional)
    - chunk_lines (int, default 3; allowed 1–5)
  Outputs:
    - Full mode: return English Markdown as file attachment
      - Content-Type: text/markdown; charset=utf-8
      - Content-Disposition: attachment; filename="<name>.md"
    - Incremental mode: return JSON array of {zh, en} (or your chosen format) containing only changed lines.
  Errors:
    - 400 missing/invalid inputs
    - 404 incremental requested but repo file missing
    - 502 upstream ModelArk error
    - 500 unexpected error

- GET /healthz -> {"status":"ok"}

**Note** Return clear status of the translation task both on the UI and via API, especially design error messages that can indicate informatively.

(Optional) POST /api/preview -> JSON output for quick preview.

============================================================
4) Markdown safety preprocessing
============================================================
- Before translation, protect:
  - fenced code blocks and inline code (placeholders + restore)
  - link targets / bare URLs (URL must remain unchanged; placeholders + restore)
- during preprocessing, replace codeblocks and html markups with placeholders
- Do NOT do any placeholder replacement for glossary terms.
- After translation, restore placeholders and validate:
  - code fence balance unchanged
  - no unreplaced placeholders remain

============================================================
5) Version repository (server-side)
============================================================
- Implement save/load/replace by repository_path.
- Store under a dedicated data directory; sanitize repository_path to avoid traversal (reject .., absolute paths, etc).
- Keep repository module abstract so it can later be extended to store translation memory (lightweight DB).

============================================================
6) Debug system
============================================================
- Implement a debug system that can clearly check the log of each request


============================================================
7) Deliverables
============================================================
Provide a working repo with:
- FastAPI app + HTML UI
- ModelArk client
- Markdown protection utilities
- Glossary CSV parser + “glossary-to-system-prompt” builder
- Version repository module
- Diff logic for incremental mode
- README: setup, env vars, run command (uvicorn --host 0.0.0.0 --port $PORT), LAN access, curl examples
- examples/ sample md + glossary
- basic tests or a manual test script
Implement it.