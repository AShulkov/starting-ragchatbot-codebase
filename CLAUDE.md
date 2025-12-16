# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

**Start the server:**
```bash
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Install dependencies:**
```bash
uv sync
```

**Required environment variable:**
Create `.env` file in root with:
```
ANTHROPIC_API_KEY=your_key_here
```

**Access points:**
- Web interface: http://localhost:8000
- API docs: http://localhost:8000/docs

## Architecture Overview

This is a RAG (Retrieval-Augmented Generation) system for querying course materials using semantic search and AI responses.

### Request Flow

```
User Query → FastAPI → RAGSystem → AIGenerator (Claude)
                ↓                        ↓
          SessionManager            ToolManager
                                         ↓
                                  CourseSearchTool
                                         ↓
                                   VectorStore (ChromaDB)
```

### Core Components

**RAGSystem** (`backend/rag_system.py`)
- Main orchestrator coordinating all components
- Initializes DocumentProcessor, VectorStore, AIGenerator, SessionManager, ToolManager
- Entry point: `query(query, session_id)` returns `(response, sources)`

**AIGenerator** (`backend/ai_generator.py`)
- Interfaces with Anthropic Claude API
- Implements tool-calling pattern for search
- Key: Uses static system prompt to define AI behavior and tool usage rules
- Two-step flow: initial request → tool execution → final response with results

**ToolManager & CourseSearchTool** (`backend/search_tools.py`)
- Tool-based architecture following Anthropic's tool use pattern
- `CourseSearchTool` wraps VectorStore search with tool definition
- AI decides when to call `search_course_content` tool
- Tool tracks sources from searches for UI display
- Pattern: Tool returns formatted results, ToolManager extracts sources

**VectorStore** (`backend/vector_store.py`)
- Uses ChromaDB with Sentence Transformers embeddings
- Two collections:
  - `course_catalog`: Course metadata for fuzzy course name matching
  - `course_content`: Actual searchable content chunks
- Smart search: resolves partial course names via semantic search before content search
- Filtering: supports course_title and/or lesson_number filters

**DocumentProcessor** (`backend/document_processor.py`)
- Parses course files with expected format (Course Title/Link/Instructor, then Lesson N: Title)
- Sentence-aware chunking: 800 chars per chunk, 100 char overlap
- Adds contextual prefixes to chunks: "Course X Lesson N content: ..."
- Returns `(Course, List[CourseChunk])` tuple

**SessionManager** (`backend/session_manager.py`)
- Maintains conversation history per session
- Limits history to MAX_HISTORY exchanges (default: 2)

### Data Models

All in `backend/models.py`:
- **Course**: title, course_link, instructor, lessons[]
- **Lesson**: lesson_number, title, lesson_link
- **CourseChunk**: content, course_title, lesson_number, chunk_index

### Startup Behavior

On app startup (`app.py:startup_event`):
1. Checks for `docs/` folder
2. Calls `rag_system.add_course_folder(docs_path, clear_existing=False)`
3. Processes only NEW courses (checks existing titles in vector store)
4. This prevents duplicate processing on server restart

### Configuration

All settings in `backend/config.py`:
- `CHUNK_SIZE = 800` - Characters per chunk
- `CHUNK_OVERLAP = 100` - Overlap between chunks
- `MAX_RESULTS = 5` - Search results to return
- `MAX_HISTORY = 2` - Conversation turns to remember
- `EMBEDDING_MODEL = "all-MiniLM-L6-v2"`
- `ANTHROPIC_MODEL = "claude-sonnet-4-20250514"`
- `CHROMA_PATH = "./chroma_db"`

## Key Implementation Patterns

### Adding New Tools

1. Create tool class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in RAGSystem: `self.tool_manager.register_tool(YourTool())`

### Modifying Search Behavior

Search logic is in `VectorStore.search()`:
1. Resolves course name via semantic search on `course_catalog`
2. Builds ChromaDB filter dict
3. Queries `course_content` collection with filters
4. Returns `SearchResults` object

### Changing AI Behavior

Modify `AIGenerator.SYSTEM_PROMPT` - this defines:
- When to use search tool
- Response style and format
- Tool usage constraints (e.g., "one search per query maximum")

### Document Processing

Add support for new document formats in `DocumentProcessor.read_file()` and `DocumentProcessor.process_course_document()`. Current format expects:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content]
```

## ChromaDB Persistence

- Database stored at `./chroma_db/` (relative to where server runs)
- Data persists across server restarts
- Clear all data: `vector_store.clear_all_data()`
- Check existing courses: `vector_store.get_existing_course_titles()`

## Frontend Integration

Static files in `frontend/` directory served by FastAPI:
- `index.html` - Chat interface
- `script.js` - API calls to `/api/query` and `/api/courses`
- `style.css` - Styling

API contract:
- POST `/api/query`: `{query: str, session_id?: str}` → `{answer: str, sources: str[], session_id: str}`
- GET `/api/courses`: → `{total_courses: int, course_titles: str[]}`
- Always use camelback naming