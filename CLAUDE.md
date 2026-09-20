# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Uses `uv` and Python 3.13+. On Windows, the README says to run shell scripts from Git Bash.

```bash
uv sync                                   # install dependencies
cp .env.example .env                      # then set ANTHROPIC_API_KEY (must be in the project root)
./run.sh                                  # start the app
cd backend && uv run uvicorn app:app --reload --port 8000   # equivalent manual start
```

- **Always use `uv` to run the server and to manage all dependencies. Never use `pip` directly.** Install with `uv sync`, add with `uv add <pkg>`, remove with `uv remove <pkg>`, and run anything with `uv run`. This keeps `pyproject.toml` and `uv.lock` in sync, so don't hand-edit the dependency list.
- App: http://localhost:8000, Swagger docs: http://localhost:8000/docs
- **The server must be started from `backend/`.** Paths are relative to the working directory: `../docs`, `../frontend`, `./chroma_db`.
- There are no tests, linter or build step configured. The root `main.py` is an unused stub, not the entry point.
- The first start downloads the `all-MiniLM-L6-v2` embedding model and ingests `docs/` into `backend/chroma_db/` (gitignored).

## Architecture

A course-materials RAG chatbot. FastAPI serves both the JSON API and the static `frontend/` (plain HTML/JS, no build). Claude answers using a **tool-calling** search over ChromaDB, rather than injecting retrieved context into the prompt up front.

`backend/rag_system.py` (`RAGSystem`) wires everything together. Its collaborators are `DocumentProcessor`, `VectorStore`, `AIGenerator`, `SessionManager` and a `ToolManager` with `CourseSearchTool` registered. `app.py` builds one module-level `RAGSystem` at import time.

### Query flow (`POST /api/query`)
1. `app.py` creates a session if none is given, then calls `RAGSystem.query`.
2. `AIGenerator.generate_response` makes Claude call #1 with the `search_course_content` tool.
3. If `stop_reason == "tool_use"`, `_handle_tool_execution` runs the tool through `ToolManager` and `CourseSearchTool`, then `VectorStore.search`. It makes **call #2 without tools** for the final answer. This is a single round only, which is why the system prompt says "one search per query", so multi-search or multi-turn tool use needs changes in `ai_generator.py`.
4. Sources shown in the UI are a side channel. `CourseSearchTool` stores `last_sources` while formatting results, and `RAGSystem.query` reads and resets them via `ToolManager`. They are not part of Claude's output.

### Vector store (`vector_store.py`)
Two ChromaDB collections:
- `course_catalog`: one entry per course. The ID and document are the course title, and the metadata holds instructor, link and `lessons_json`. It is used to fuzzy-resolve a `course_name` argument to a real title by taking the nearest match (`n_results=1`), so it always resolves to something.
- `course_content`: text chunks, with metadata `course_title`, `lesson_number` and `chunk_index`. These fields drive the search filters.

The course title is the identity everywhere: it is the catalog ID, the chunk metadata key, and the basis of the chunk IDs (`<Title_with_underscores>_<chunk_index>`).

### Ingestion (`document_processor.py`)
On startup, `app.py` runs `add_course_folder("../docs")`. It skips any course whose title already exists in the catalog, so **editing a transcript without changing its title will not re-ingest it**. Use `add_course_folder(..., clear_existing=True)`, or delete `backend/chroma_db/`, to rebuild. Nothing in the API exposes that.

Documents must follow this format, or the parser falls back to treating the whole file as one un-numbered document:
```
Course Title: ...
Course Link: ...
Course Instructor: ...

Lesson 0: <title>
Lesson Link: ...
<transcript text>
Lesson 1: ...
```
Text is chunked by sentence (`CHUNK_SIZE=800`, `CHUNK_OVERLAP=100` characters) and each chunk is prefixed with lesson context. `.pdf` and `.docx` files are accepted by the folder scan, but `read_file` reads everything as UTF-8 text, so only `.txt` actually works.

### Configuration and state
- `config.py` holds all tunables (model, chunk sizes, `MAX_RESULTS=5`, `MAX_HISTORY=2`, `CHROMA_PATH`) as a dataclass, and loads `ANTHROPIC_API_KEY` from `.env`.
- Sessions are in-memory only (`SessionManager`), so they are lost on restart. History is passed to Claude as text appended to the system prompt, not as message turns.
- The frontend starts a new session only on page load (`currentSessionId = null`).
