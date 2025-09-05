# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend<br/>(script.js)
    participant FastAPI as FastAPI<br/>(app.py)
    participant RAGSystem as RAG System<br/>(rag_system.py)
    participant SessionMgr as Session Manager<br/>(session_manager.py)
    participant AIGen as AI Generator<br/>(ai_generator.py)
    participant Claude as Anthropic Claude<br/>(API)
    participant ToolMgr as Tool Manager<br/>(search_tools.py)
    participant VectorDB as Vector Store<br/>(ChromaDB)

    %% User Input Phase
    User->>Frontend: Types query & clicks send
    Frontend->>Frontend: Disable input, show loading
    Frontend->>Frontend: Add user message to chat

    %% API Request Phase
    Frontend->>FastAPI: POST /api/query<br/>{query, session_id}
    FastAPI->>FastAPI: Create session if needed
    FastAPI->>RAGSystem: query(request.query, session_id)

    %% RAG Processing Phase
    RAGSystem->>SessionMgr: get_conversation_history(session_id)
    SessionMgr-->>RAGSystem: Previous messages context
    RAGSystem->>AIGen: generate_response(query, history, tools, tool_manager)

    %% AI Generation Phase
    AIGen->>Claude: messages.create() with system prompt & tools
    Claude->>Claude: Analyze query - need search?
    
    alt Query needs course content search
        Claude-->>AIGen: tool_use response
        AIGen->>ToolMgr: execute_tool("search", query_params)
        ToolMgr->>VectorDB: semantic_search(query)
        VectorDB-->>ToolMgr: relevant chunks + metadata
        ToolMgr-->>AIGen: search results
        AIGen->>Claude: messages.create() with tool results
        Claude-->>AIGen: synthesized response
    else General knowledge query
        Claude-->>AIGen: direct response
    end

    %% Response Assembly Phase
    AIGen-->>RAGSystem: response text
    RAGSystem->>ToolMgr: get_last_sources()
    ToolMgr-->>RAGSystem: sources list
    RAGSystem->>SessionMgr: add_exchange(session_id, query, response)
    RAGSystem-->>FastAPI: (response, sources)

    %% API Response Phase
    FastAPI-->>Frontend: JSON{answer, sources, session_id}
    Frontend->>Frontend: Remove loading animation
    Frontend->>Frontend: Display response with sources
    Frontend->>Frontend: Re-enable input
    Frontend->>User: Show answer in chat
```

## Key Components:

### Frontend Layer
- **User Interface**: Chat input, loading states, message display
- **Session Management**: Tracks currentSessionId for conversation continuity
- **API Communication**: Handles HTTP requests/responses

### Backend API Layer  
- **FastAPI Router**: `/api/query` endpoint with request validation
- **Error Handling**: HTTP exceptions and status codes
- **Response Formatting**: Structured JSON responses

### RAG Core Layer
- **Orchestration**: Coordinates all system components
- **Tool Integration**: Manages search tool availability
- **Context Assembly**: Combines query, history, and tools

### AI Processing Layer
- **Claude Integration**: Anthropic API calls with tool support
- **Tool Execution**: Handles tool_use responses automatically
- **Response Synthesis**: Combines search results into coherent answers

### Data Layer
- **Vector Database**: ChromaDB for semantic search
- **Session Storage**: Conversation history persistence
- **Document Processing**: Pre-chunked course materials

## Flow Characteristics:
- **Intelligent Tool Usage**: Claude decides when to search vs. use general knowledge
- **Context Preservation**: Session management maintains conversation history  
- **Semantic Search**: Vector similarity matching for relevant content retrieval
- **Error Resilience**: Graceful handling of failures at each layer
- **Real-time UI**: Loading states and progressive disclosure of results