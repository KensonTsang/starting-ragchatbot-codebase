# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RAG chatbot that answers questions about course transcripts. FastAPI backend, ChromaDB vector store, Anthropic Claude for generation, and a static HTML/JS frontend served by the same FastAPI process.

## Commands

Dependencies are managed with `uv` (Python 3.13+, pinned in `.python-version`).

```bash
uv sync                      # install dependencies
./run.sh                     # start the server on port 8000
cd backend && uv run uvicorn app:app --reload --port 8000   # equivalent manual start
```

- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs
- Requires `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`).

There is no test suite, linter, or formatter configured.

## Working directory matters

The server must be started from `backend/`. Three paths are relative to it:

- `../docs`: course documents ingested at startup (`app.py`)
- `../frontend`: static files mounted at `/` (`app.py`)
- `./chroma_db`: persistent vector store (`config.py`), gitignored

Backend modules import each other by bare name (`from config import config`), not as a package, for the same reason.

## Architecture

`RAGSystem` (`backend/rag_system.py`) is the single orchestrator. `app.py` creates one instance at import time and both API endpoints go through it. It owns a `DocumentProcessor`, `VectorStore`, `AIGenerator`, `SessionManager`, and a `ToolManager` with one registered tool.

### Retrieval is tool-driven, not always-on

The backend does not search before calling Claude. `AIGenerator.generate_response` sends the question with the `search_course_content` tool and `tool_choice: auto`, and Claude decides whether to search.

- If Claude answers directly, that is the response (one API call).
- If Claude returns `tool_use`, `_handle_tool_execution` runs the tool, then makes a second API call **without tools**. This caps retrieval at one search round per question; the system prompt also says "one search per query maximum".

Conversation history is injected into the system prompt as a formatted string, not as message turns. The `messages` list always starts with a single user message.

### Two ChromaDB collections

`VectorStore` keeps two collections, both embedded with `all-MiniLM-L6-v2`:

- `course_catalog`: one entry per course. The embedded document is the course title, and the ID is also the title. Lessons are stored as a JSON string in the `lessons_json` metadata field because Chroma metadata must be flat.
- `course_content`: the text chunks, with `course_title`, `lesson_number`, and `chunk_index` metadata.

`VectorStore.search` uses the catalog to turn a fuzzy `course_name` from Claude (for example "MCP") into an exact title via a top-1 vector query, then filters `course_content` on that title and/or lesson number. There is no distance threshold, so any course name resolves to the nearest title.

### Sources travel out of band

`CourseSearchTool` stores the source labels for its last search on `self.last_sources`. After generation, `RAGSystem.query` reads them through `ToolManager.get_last_sources()` and then resets them. This state lives on a single shared tool instance, so it is not safe under concurrent requests.

### Adding a tool

Subclass `Tool` in `backend/search_tools.py`, implement `get_tool_definition()` (Anthropic tool schema) and `execute(**kwargs)`, and register it with `tool_manager.register_tool(...)` in `RAGSystem.__init__`. To surface sources in the UI, give the tool a `last_sources` list attribute.

## Document format and ingestion

Files in `docs/` are parsed by `DocumentProcessor.process_course_document`, which expects:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<transcript text>

Lesson 1: <lesson title>
...
```

Lesson text is split into sentence-based chunks (`CHUNK_SIZE` 800 characters, `CHUNK_OVERLAP` 100, both in `config.py`).

Things to know before changing ingestion:

- **The course title is the unique key.** At startup every file is re-parsed, but a course whose title already exists in `course_catalog` is skipped. Edits to an existing document are not picked up until `backend/chroma_db/` is deleted or `add_course_folder(..., clear_existing=True)` is used.
- **Chunk prefixes are inconsistent.** For most lessons only the first chunk gets a `Lesson N content:` prefix; for the last lesson of a course every chunk gets `Course <title> Lesson N content:`.
- **Only plain text is supported.** The folder loader accepts `.pdf` and `.docx` extensions, but `read_file` opens every file as UTF-8 text.

## Other behaviours

- Sessions are held in memory by `SessionManager` and are lost on restart. `MAX_HISTORY` (2) is the number of exchanges kept.
- `/api/query` is declared `async` but calls the synchronous Anthropic and Chroma clients, so a slow request blocks the event loop.
- The frontend has no build step. `index.html` loads `script.js` and `style.css` with a `?v=` query string for cache busting; bump it when changing those files.
- `main.py` at the repo root is an unused stub. The real entry point is `backend/app.py`.
- `DevStaticFiles` in `app.py` (no-cache headers) is defined but not used; the mount uses plain `StaticFiles`.
