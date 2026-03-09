# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Quick start (from repo root)
./run.sh

# Manual start (must run from backend/ directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

The server runs at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

**Requirements:** Set `ANTHROPIC_API_KEY` in a `.env` file at the repo root (see `.env.example`).

## Architecture

This is a tool-based RAG system where Claude decides when to search, rather than always pre-fetching context.

**Request flow:**
1. `POST /api/query` → `RAGSystem.query()` builds a prompt + retrieves session history
2. `AIGenerator.generate_response()` calls Claude with a `search_course_content` tool available
3. If Claude calls the tool, `CourseSearchTool.execute()` → `VectorStore.search()` → ChromaDB semantic search
4. Claude synthesizes a final answer from tool results; sources are tracked and returned to the UI

**On startup**, `app.py` loads all `.txt` files from `../docs/` into ChromaDB (skips already-loaded courses by title).

## Key Architectural Decisions

- **Tool-based retrieval**: Claude autonomously decides whether to search (course-specific questions) or answer directly (general knowledge). Capped at one search per query via the system prompt.
- **Two ChromaDB collections**: `course_catalog` stores course-level metadata (title, instructor, links, lessons as JSON); `course_content` stores chunked lesson text for semantic search.
- **Course title as ID**: `Course.title` is the unique identifier used as the ChromaDB document ID in `course_catalog` and as a metadata filter key in `course_content`.
- **Semantic course name resolution**: When Claude passes a `course_name` filter, `VectorStore._resolve_course_name()` does a vector search in `course_catalog` to fuzzy-match partial names to exact titles.
- **Session history as string**: Conversation history is formatted as a plain string injected into the system prompt (not as structured messages). Stored in-memory only; lost on server restart.

## Course Document Format

Documents in `docs/` must follow this structure for `DocumentProcessor` to parse them correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <lesson title>
...
```

The title string is used as the unique course ID — changing it creates a duplicate entry.

## Configuration

All tunable parameters live in `backend/config.py` as a `Config` dataclass:
- `CHUNK_SIZE` / `CHUNK_OVERLAP`: Controls text chunking (default 800/100 chars)
- `MAX_RESULTS`: Number of ChromaDB results returned per search (default 5)
- `MAX_HISTORY`: Conversation turns retained per session (default 2)
- `ANTHROPIC_MODEL`: Claude model used for generation
- `CHROMA_PATH`: Where ChromaDB persists data (default `./chroma_db`, relative to `backend/`)

## Adding New Tools

Tools are registered with `ToolManager` and passed to Claude as Anthropic tool definitions. To add a new tool:
1. Subclass `Tool` (abstract base in `search_tools.py`) implementing `get_tool_definition()` and `execute()`
2. Register it: `tool_manager.register_tool(MyTool(...))`
3. If the tool tracks sources, add a `last_sources` list attribute — `ToolManager.get_last_sources()` checks for this automatically
