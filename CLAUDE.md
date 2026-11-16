# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Python 3.13, managed with `uv`. Run everything from Git Bash on Windows (`run.sh` is bash).

```bash
uv sync                                  # install dependencies
cp .env.example .env                     # then set ANTHROPIC_API_KEY
./run.sh                                 # start the server on :8000
cd backend && uv run uvicorn app:app --reload --port 8000   # equivalent, manual
```

There is no test suite, linter, or formatter configured in this repo. Don't assume `pytest` or `ruff` exist; check before recommending them.

The server must be started from `backend/`. Paths are relative to the working directory: `app.py` loads `../docs` at startup and mounts `../frontend`, and `CHROMA_PATH` is `./chroma_db`, so the vector DB lands in `backend/chroma_db/`. Running `uvicorn` from the repo root will silently skip ingestion and fail to serve the frontend.

## Architecture

The backend is one FastAPI process in `backend/`. All modules import each other by bare name (`from vector_store import ...`), so `backend/` must be on the import path. Don't add package-style imports without changing how the app is launched.

**Request path for `POST /api/query`:** `app.py` → `RAGSystem.query()` → `AIGenerator.generate_response()` → Claude → optional tool call → `CourseSearchTool` → `VectorStore.search()` → ChromaDB → back to Claude for the final answer. The flow only makes sense when read across `rag_system.py`, `ai_generator.py`, `search_tools.py`, and `vector_store.py` together.

**Claude makes at most one search, and the code enforces it.** The first `messages.create` call includes the tools. If it returns `stop_reason == "tool_use"`, `_handle_tool_execution` runs the tool and makes a second call with `tools` omitted, so Claude can't search again. Changing this means changing `ai_generator.py`, not just the system prompt. The system prompt's "one search per query" line is a second, softer enforcement.

**Two ChromaDB collections, both keyed by course title.**
- `course_catalog`: one document per course, with `id = title`. Holds the instructor, the course link, and a JSON string of lessons with their links. Used to resolve a fuzzy `course_name` to an exact title, and to look up links.
- `course_content`: one document per chunk, with metadata `course_title`, `lesson_number`, `chunk_index`. Chunk IDs are `{title with spaces→_}_{chunk_index}`.

A course's title is its identity. `add_course_folder()` skips any file whose title already exists in the catalog, so editing a course file and restarting does nothing. To re-ingest, call `add_course_folder(..., clear_existing=True)` or delete `backend/chroma_db/`. Startup currently passes `clear_existing=False`.

**Course file format** is parsed positionally in `document_processor.py`: line 1 `Course Title:`, then `Course Link:` and `Course Instructor:` within the first few lines, then `Lesson N: Title` markers, each optionally followed by a `Lesson Link:` line. Files without lesson markers fall back to a single undifferentiated chunk stream. Only `docs/*.txt` is present in the repo; `.pdf` and `.docx` are accepted by `add_course_folder` but `process_course_document` reads everything as UTF-8 text, so those formats won't parse correctly.

**Chunk context prefixes are inconsistent.** In `process_course_document`, the first chunk of a lesson inside the loop gets `Lesson N content:`, and the trailing lesson block gets `Course {title} Lesson N content:` on every chunk. Retrieval quality depends on these prefixes, so keep them in mind if you change chunking.

**Sessions are in-memory only.** `SessionManager` is a dict keyed by `session_N`, reset on every restart. History is passed to Claude as text inside the system prompt, not as message turns. `MAX_HISTORY` is a count of exchanges, stored as `2 × MAX_HISTORY` messages.

**Sources travel through shared state.** `CourseSearchTool.last_sources` is an instance attribute on a single tool object that all requests share. `RAGSystem.query()` reads it after the call and resets it. This is only correct if requests don't overlap, which the current code doesn't guarantee (see below).

**Known issues to keep in mind:**
- `query` in `app.py` is `async def` but calls the synchronous Anthropic and Chroma clients, so each query blocks the event loop.
- `VectorStore.get_lesson_link()` has no `return` on its exception path, so it returns `None` implicitly. Nothing currently calls it.
- `DevStaticFiles` is defined in `app.py` but not used. The mount uses plain `StaticFiles`, so no-cache headers are not sent.
- `main.py` is an unused placeholder.

**Adding a tool:** implement the `Tool` ABC in `search_tools.py` (`get_tool_definition`, `execute`) and register it in `RAGSystem.__init__`. If it should report sources to the UI, it must set a `last_sources` attribute, because `ToolManager` finds sources by checking for that attribute.

**Configuration** lives in `backend/config.py` as a dataclass with defaults; only `ANTHROPIC_API_KEY` is read from the environment. `ANTHROPIC_MODEL` is hardcoded. `.env` is loaded by `load_dotenv()` from the project root (`backend/.env` also works, as `.gitignore` shows).

**Frontend** is static files in `frontend/`, served by the same FastAPI app. It calls `/api/query` and `/api/courses` with relative URLs, so the frontend and API must share an origin. Sending a message is `sendMessage()` in `script.js`, which keeps `currentSessionId` from the first response.

## Existing docs

`README.md` covers installation and running. There are no Cursor, Copilot, Codex, or Gemini configuration files in this repo.
