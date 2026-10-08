# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
uv sync                                                # install dependencies (Python 3.13, uv-managed .venv)
./run.sh                                               # start the app (Git Bash on Windows)
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Always use `uv` (`uv sync`, `uv add`, `uv run ...`) to manage dependencies and run code; never use `pip` or invoke `python` directly.
- The server must be started from `backend/`: it loads documents from `../docs` and persists ChromaDB to `./chroma_db` via relative paths.
- Web UI at `http://localhost:8000`, API docs at `/docs`. Smoke test: `curl -X POST localhost:8000/api/query -H "Content-Type: application/json" -d '{"query":"..."}'`.
- `ANTHROPIC_API_KEY` must be in the environment or in `.env` at the repo root (see `.env.example`). Without it the server starts but every query returns 500.
- On Windows, `--reload` can hang after a code change and keep serving the old code; restart the process manually if changes don't take effect. `WinError 10013` on startup means port 8000 is already taken.
- There are no tests, linter, or formatter configured. `main.py` at the root is an unused placeholder.

## Architecture

FastAPI backend (`backend/`) serving a static vanilla-JS frontend (`frontend/`) and two endpoints: `POST /api/query` and `GET /api/courses`. `RAGSystem` (`rag_system.py`) wires all components together; a single instance is shared by all requests.

**Retrieval is tool-driven, not automatic.** A query is sent to Claude together with one tool, `search_course_content` (`search_tools.py`). Claude decides whether to search; the system prompt in `ai_generator.py` limits it to one search per query. If it calls the tool, `AIGenerator._handle_tool_execution` runs it and makes a second Claude call *without tools*, so only one search round is possible per question. Sources shown in the UI come from `CourseSearchTool.last_sources`, which `RAGSystem.query` reads and resets after each query.

**Two ChromaDB collections** (`vector_store.py`, embeddings from `all-MiniLM-L6-v2`):
- `course_catalog`: one entry per course; the embedded text is just the title. Used to resolve a fuzzy `course_name` (e.g. "MCP") to an exact title through top-1 semantic search.
- `course_content`: text chunks with `course_title` / `lesson_number` metadata, filtered by the resolved title and lesson.

The course title is the unique ID everywhere (catalog ID, filters, chunk IDs `<title>_<chunk_index>`).

**Ingestion runs on every server startup** (`app.py` startup → `add_course_folder`). Files in `docs/` are parsed by `document_processor.py` and expect this format: `Course Title:` / `Course Link:` / `Course Instructor:` header lines, then `Lesson N: Title` markers, each optionally followed by a `Lesson Link:` line. Courses whose title already exists in ChromaDB are skipped, so edited documents are not re-indexed; delete `backend/chroma_db/` to rebuild. `.pdf`/`.docx` pass the extension filter but are read as plain UTF-8 text, so only `.txt` actually works.

**Conversation history** (`session_manager.py`) is in memory only (lost on restart, keeps `MAX_HISTORY` exchanges) and is passed to Claude as text appended to the system prompt, not as message turns.

## Claude API constraints

The model is set in `backend/config.py` (`claude-sonnet-5-5`). That model rejects a non-default `temperature` and may return `thinking` blocks, so `ai_generator.py`:
- sends `thinking: {"type": "between_tools"}` (lowest thinking setting) and no `temperature`;
- reads only `text` blocks from responses (`_extract_text`), never `content[0]`;
- drops `thinking` blocks from the assistant turn before the follow-up call, because that call omits `tools` and replaying thinking blocks after such a change can be rejected.

Keep these in mind when changing the model or the request parameters. The `anthropic` SDK is pinned to an old version (`0.58.2`); newer request fields are passed as plain dicts.

## Gotchas

- The frontend (`frontend/script.js`) shows only a generic "Query failed" for any non-2xx response; the real error is in the response `detail` field (check it with curl).
- `/api/query` is `async def` but calls blocking code (Anthropic SDK, ChromaDB, embeddings), so queries are serialized and block the event loop.
- Chunk context prefixes in `document_processor.py` are inconsistent: the last lesson of a course gets `Course <title> Lesson N content:` on every chunk, while other lessons only get `Lesson N content:` on their first chunk.
