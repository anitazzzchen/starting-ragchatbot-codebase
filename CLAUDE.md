# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

### Running the Application
```bash
# Quick start
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Development
```bash
# Install/sync dependencies
uv sync

# Access points
# Web interface: http://localhost:8000
# API docs: http://localhost:8000/docs
```

### Environment Setup
Required `.env` file in root:
```
ANTHROPIC_API_KEY=your_api_key_here
```

## Architecture Overview

This is a Retrieval-Augmented Generation (RAG) system for course materials with a modular, tool-based architecture.

### Core Architecture Pattern

The system follows a **tool-based RAG architecture** where Claude intelligently decides when to search the knowledge base versus using general knowledge:

1. **Frontend** (`frontend/`) - Web chat interface communicating via `/api/query`
2. **RAG Orchestrator** (`backend/rag_system.py`) - Coordinates all components
3. **AI Layer** (`backend/ai_generator.py`) - Handles Claude API with tool execution
4. **Tool System** (`backend/search_tools.py`) - Provides semantic search capabilities
5. **Vector Storage** (`backend/vector_store.py`) - ChromaDB with dual collections
6. **Document Processing** (`backend/document_processor.py`) - Structured text chunking

### Key Architectural Patterns

**Tool-Based Decision Making**: Claude receives a search tool and decides autonomously whether to use it based on query context. This avoids unnecessary searches for general knowledge questions.

**Dual Vector Storage**: 
- `course_content` collection: Chunked course material with lesson context
- `course_metadata` collection: Course/lesson metadata for filtering

**Session-Based Context**: Conversation history maintained via `SessionManager` with configurable limits (`MAX_HISTORY`).

**Smart Document Structure**: Course documents parsed with metadata (title, instructor, link) and lessons automatically detected via regex patterns (`Lesson X: Title`).

**Context-Enhanced Chunking**: Chunks prefixed with course/lesson context for better retrieval relevance.

### Data Models

Three core Pydantic models in `backend/models.py`:
- `Course`: Metadata with lessons list
- `Lesson`: Individual lesson with optional link
- `CourseChunk`: Text chunk with hierarchical references

### Configuration

Centralized config in `backend/config.py`:
- Chunk size/overlap for text processing
- Embedding model (`all-MiniLM-L6-v2`)
- Claude model (`claude-sonnet-4-20250514`) 
- ChromaDB path and search limits

### Document Processing Flow

1. Parse metadata from first 3 lines (title, link, instructor)
2. Detect lesson boundaries with `Lesson X: Title` pattern
3. Chunk text using sentence-based splitting with overlap
4. Enhance chunks with contextual prefixes
5. Store in vector database with metadata

### Query Processing Flow

1. Frontend sends query to `/api/query`
2. RAG system retrieves conversation history
3. AI generator calls Claude with search tool available
4. Claude autonomously decides to search or answer directly
5. If searching: Vector similarity search returns relevant chunks
6. Claude synthesizes response from search results
7. Session updated with query/response pair

## Development Notes

### Adding New Tools
Implement `Tool` abstract base class in `search_tools.py`. Register with `ToolManager` in `rag_system.py`.

### Vector Store Collections
- Use `add_course_content()` for searchable text chunks
- Use `add_course_metadata()` for course/lesson metadata
- Both support filtering and semantic search

### Frontend-Backend Contract
API expects `QueryRequest` with `query` and optional `session_id`. Returns `QueryResponse` with `answer`, `sources`, and `session_id`.

### Document Format
Course files should follow this structure:
```
Course Title: [title]
Course Link: [url] 
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [url]
[lesson content...]

Lesson 1: Next Topic
[lesson content...]
```
- always use uv to run the server do not use pip directly
- make sure to use uv to manage all dependencies