# Dify Architecture Diagrams

This directory contains visual architecture diagrams for the Dify project.

## Available Diagrams

### 1. System Architecture
See the complete system architecture in:
- [English Version](../ARCHITECTURE.md)
- [中文版本](../zh-CN/ARCHITECTURE.md)

### 2. Mermaid Diagrams

All diagrams are created using Mermaid syntax and can be viewed in:
- GitHub (native support)
- VS Code (with Mermaid extension)
- Online Mermaid editors

### 3. Component Interaction

```mermaid
graph TB
    subgraph "User Layer"
        U[End User]
        A[Admin User]
    end
    
    subgraph "Presentation Layer"
        WEB[Next.js Web UI]
        WEB_CONSOLE[Console Interface]
        WEB_APP[App Interface]
        WEB --> WEB_CONSOLE
        WEB --> WEB_APP
    end
    
    subgraph "API Layer"
        NGINX[Nginx Gateway]
        API_CONSOLE[Console API]
        API_PUBLIC[Public API]
        NGINX --> API_CONSOLE
        NGINX --> API_PUBLIC
    end
    
    subgraph "Service Layer"
        APP_SVC[App Service]
        DATASET_SVC[Dataset Service]
        MODEL_SVC[Model Service]
        WORKFLOW_SVC[Workflow Service]
        AGENT_SVC[Agent Service]
    end
    
    subgraph "Core Layer"
        MODEL_MGR[Model Manager]
        WORKFLOW_ENG[Workflow Engine]
        RAG_PIPELINE[RAG Pipeline]
        AGENT_RUNTIME[Agent Runtime]
        PROMPT_ENG[Prompt Engine]
    end
    
    subgraph "Data Access Layer"
        REPO[Repositories]
        CACHE[Cache Manager]
        QUEUE[Task Queue]
    end
    
    subgraph "Storage Layer"
        DB[(Database)]
        REDIS[(Redis)]
        VECTOR[(Vector DB)]
        STORAGE[(Object Storage)]
    end
    
    subgraph "External Services"
        LLM[LLM Providers]
        TOOLS[External Tools]
    end
    
    U --> WEB_APP
    A --> WEB_CONSOLE
    WEB --> NGINX
    API_CONSOLE --> APP_SVC
    API_CONSOLE --> DATASET_SVC
    API_CONSOLE --> MODEL_SVC
    API_PUBLIC --> APP_SVC
    
    APP_SVC --> WORKFLOW_SVC
    APP_SVC --> AGENT_SVC
    DATASET_SVC --> RAG_PIPELINE
    MODEL_SVC --> MODEL_MGR
    WORKFLOW_SVC --> WORKFLOW_ENG
    AGENT_SVC --> AGENT_RUNTIME
    
    WORKFLOW_ENG --> MODEL_MGR
    AGENT_RUNTIME --> MODEL_MGR
    RAG_PIPELINE --> VECTOR
    MODEL_MGR --> LLM
    AGENT_RUNTIME --> TOOLS
    
    APP_SVC --> REPO
    DATASET_SVC --> REPO
    WORKFLOW_SVC --> QUEUE
    
    REPO --> DB
    CACHE --> REDIS
    QUEUE --> REDIS
    RAG_PIPELINE --> STORAGE
    
    style U fill:#e1f5ff
    style A fill:#e1f5ff
    style WEB fill:#d4edda
    style NGINX fill:#fff3cd
    style DB fill:#f8d7da
    style REDIS fill:#f8d7da
    style VECTOR fill:#f8d7da
    style STORAGE fill:#f8d7da
    style LLM fill:#e7d4f5
    style TOOLS fill:#e7d4f5
```

### 4. Data Flow - RAG Query

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Web as Web Frontend
    participant API as Flask API
    participant Cache as Redis Cache
    participant Vector as Vector DB
    participant LLM as LLM Provider
    participant DB as PostgreSQL
    
    User->>Web: Send query message
    Web->>API: POST /chat-messages
    API->>Cache: Check conversation context
    Cache-->>API: Return context (if exists)
    
    API->>DB: Load app configuration
    DB-->>API: Return config
    
    API->>Vector: Search relevant documents
    Vector-->>API: Return top-k chunks
    
    API->>API: Assemble prompt with context
    API->>LLM: Stream completion request
    
    loop Streaming response
        LLM-->>API: Token chunk
        API-->>Web: SSE stream
        Web-->>User: Display token
    end
    
    LLM-->>API: Complete response
    API->>DB: Save message & response
    API->>Cache: Update conversation context
    API-->>Web: Final message metadata
```

### 5. Data Flow - Document Indexing

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Web as Web Frontend
    participant API as Flask API
    participant Storage as Object Storage
    participant Queue as Redis Queue
    participant Worker as Celery Worker
    participant Vector as Vector DB
    participant LLM as LLM Provider
    participant DB as PostgreSQL
    
    User->>Web: Upload document
    Web->>API: POST /datasets/{id}/documents
    API->>API: Validate file
    API->>Storage: Save original file
    Storage-->>API: Return file URL
    
    API->>DB: Create document record
    DB-->>API: Return document ID
    
    API->>Queue: Enqueue indexing task
    API-->>Web: Return document ID & status=queued
    Web-->>User: Show upload success
    
    Queue->>Worker: Dispatch indexing task
    Worker->>Storage: Download file
    Storage-->>Worker: Return file content
    
    Worker->>Worker: Parse document
    Worker->>Worker: Split into chunks
    
    loop For each chunk
        Worker->>LLM: Generate embedding
        LLM-->>Worker: Return vector
        Worker->>Vector: Store chunk + vector
    end
    
    Worker->>DB: Update document status=completed
    Worker->>Queue: Acknowledge task completion
    
    Note over User,DB: User can now query the document
```

### 6. Workflow Execution

```mermaid
stateDiagram-v2
    [*] --> Initialized: Create Workflow Run
    Initialized --> Running: Start Execution
    
    Running --> NodeExecuting: Execute Node
    NodeExecuting --> NodeSuccess: Node Completed
    NodeExecuting --> NodeFailed: Node Error
    
    NodeSuccess --> HasNextNode: Check Flow
    NodeFailed --> ErrorHandling: Handle Error
    
    ErrorHandling --> Retry: Can Retry
    ErrorHandling --> Failed: Max Retries
    Retry --> NodeExecuting
    
    HasNextNode --> NodeExecuting: More Nodes
    HasNextNode --> Completed: No More Nodes
    
    Failed --> [*]
    Completed --> [*]
    
    note right of NodeExecuting
        Node Types:
        - LLM Node
        - Knowledge Retrieval
        - Code Execution
        - HTTP Request
        - Conditional Branch
        - Loop
    end note
```

### 7. Agent Decision Flow

```mermaid
flowchart TD
    Start([User Query]) --> ParseQuery[Parse Query Intent]
    ParseQuery --> SelectAgent{Select Agent Type}
    
    SelectAgent -->|Function Calling| FCAgent[Function Calling Agent]
    SelectAgent -->|ReAct| ReactAgent[ReAct Agent]
    
    FCAgent --> AnalyzeTools[Analyze Available Tools]
    AnalyzeTools --> CallLLM1[Call LLM with Tools]
    CallLLM1 --> HasToolCall{Has Tool Call?}
    
    HasToolCall -->|Yes| ExecuteTool[Execute Tool]
    ExecuteTool --> ToolResult[Get Tool Result]
    ToolResult --> CallLLM1
    
    HasToolCall -->|No| FinalAnswer1[Final Answer]
    
    ReactAgent --> Thought[LLM Thinks]
    Thought --> Action{Choose Action}
    
    Action -->|Use Tool| SelectTool[Select Tool]
    Action -->|Final Answer| FinalAnswer2[Final Answer]
    
    SelectTool --> ExecuteAction[Execute Action]
    ExecuteAction --> Observation[Observe Result]
    Observation --> Thought
    
    FinalAnswer1 --> Response[Return Response]
    FinalAnswer2 --> Response
    Response --> End([Done])
    
    style Start fill:#e1f5ff
    style End fill:#d4edda
    style Response fill:#d4edda
```

### 8. Security Layers

```mermaid
graph TB
    subgraph "Network Security"
        SSL[TLS/SSL Encryption]
        CORS[CORS Policy]
        RATE[Rate Limiting]
    end
    
    subgraph "Authentication"
        JWT[JWT Tokens]
        API_KEY[API Keys]
        OAUTH[OAuth2 Integration]
    end
    
    subgraph "Authorization"
        RBAC[Role-Based Access Control]
        TENANT[Tenant Isolation]
        SCOPE[Permission Scopes]
    end
    
    subgraph "Data Protection"
        ENCRYPT[Data Encryption at Rest]
        MASK[Data Masking]
        AUDIT[Audit Logging]
    end
    
    subgraph "Application Security"
        XSS[XSS Protection]
        SSRF[SSRF Protection]
        SQL[SQL Injection Prevention]
        CSRF[CSRF Protection]
    end
    
    Request[Incoming Request] --> SSL
    SSL --> CORS
    CORS --> RATE
    RATE --> JWT
    JWT --> RBAC
    RBAC --> TENANT
    TENANT --> XSS
    XSS --> SSRF
    SSRF --> SQL
    SQL --> CSRF
    CSRF --> APP[Application Logic]
    APP --> ENCRYPT
    ENCRYPT --> AUDIT
    
    style Request fill:#fff3cd
    style APP fill:#d4edda
```

## How to View

### GitHub
All Mermaid diagrams render automatically in GitHub's web interface.

### VS Code
Install the "Markdown Preview Mermaid Support" extension:
```
ext install bierner.markdown-mermaid
```

### Online Editors
- [Mermaid Live Editor](https://mermaid.live/)
- [Mermaid Chart](https://www.mermaidchart.com/)

## Export as Images

To export diagrams as PNG/SVG:

1. Use Mermaid CLI:
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i diagram.md -o diagram.png
```

2. Use online editor:
   - Paste code into [Mermaid Live Editor](https://mermaid.live/)
   - Click "Export" button
   - Choose PNG or SVG format

## Contributing

When adding new diagrams:
1. Use consistent styling
2. Add clear labels and descriptions
3. Keep diagrams focused on specific aspects
4. Update this README with new diagram descriptions
