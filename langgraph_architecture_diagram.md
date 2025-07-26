# LangGraph Architecture Mermaid Diagrams

## 1. Overall System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        C[Client/Frontend]
        WS[WebSocket Connection]
    end
    
    subgraph "API Layer"
        API[FastAPI Server]
        CR[Chat Router]
        WSR[WebSocket Router]
        TC[Tool Confirmation]
    end
    
    subgraph "Service Layer"
        CS[Chat Service]
        LGS[LangGraph Service]
        TS[Tool Service]
        WSS[WebSocket Service]
        DBS[Database Service]
        CFGS[Config Service]
    end
    
    subgraph "LangGraph Multi-Agent System"
        direction TB
        AM[Agent Manager]
        SWARM[LangGraph Swarm]
        
        subgraph "Agents"
            PA[Planner Agent]
            IVCA[Image/Video Creator Agent]
        end
        
        subgraph "Stream Processing"
            SP[Stream Processor]
            SC[Stream Chunks]
        end
    end
    
    subgraph "Tool Ecosystem"
        direction LR
        subgraph "System Tools"
            WPT[Write Plan Tool]
            CFT[ComfyUI Tools]
        end
        
        subgraph "Image Generation Tools"
            GPT[GPT Image 1]
            IMG4[Imagen 4]
            FLUX[Flux Series]
            RC[Recraft v3]
            MJ[Midjourney]
        end
        
        subgraph "Video Generation Tools"
            SD[Seedance v1]
            KL[Kling v2]
            HL[Hailuo 02]
            VEO[Veo3 Fast]
        end
    end
    
    subgraph "External Providers"
        JAAZ[JAAZ API]
        REP[Replicate API]
        OAI[OpenAI API]
        ANT[Anthropic API]
        OLL[Ollama Local]
    end
    
    subgraph "Data Layer"
        DB[(SQLite Database)]
        CFG[Configuration Files]
    end
    
    %% Client connections
    C <--> WS
    WS <--> WSR
    C --> API
    
    %% API routing
    API --> CR
    API --> WSR
    API --> TC
    
    %% Service layer connections
    CR --> CS
    WSR --> WSS
    CS --> LGS
    CS --> DBS
    
    %% LangGraph system
    LGS --> AM
    AM --> SWARM
    SWARM --> PA
    SWARM --> IVCA
    SWARM --> SP
    SP --> SC
    
    %% Tool connections
    PA --> WPT
    IVCA --> GPT
    IVCA --> IMG4
    IVCA --> FLUX
    IVCA --> RC
    IVCA --> MJ
    IVCA --> SD
    IVCA --> KL
    IVCA --> HL
    IVCA --> VEO
    
    %% Tool service management
    TS --> WPT
    TS --> GPT
    TS --> IMG4
    TS --> FLUX
    
    %% External provider connections
    GPT --> JAAZ
    IMG4 --> JAAZ
    FLUX --> REP
    RC --> REP
    SD --> JAAZ
    
    %% LLM connections
    PA --> OAI
    PA --> ANT
    PA --> OLL
    IVCA --> OAI
    IVCA --> ANT
    IVCA --> OLL
    
    %% Data persistence
    DBS --> DB
    CFGS --> CFG
    
    %% Real-time communication
    SP --> WSS
    WSS --> WS
    
    style SWARM fill:#e1f5fe
    style PA fill:#f3e5f5
    style IVCA fill:#e8f5e8
    style SP fill:#fff3e0
```

## 2. End-to-End Execution Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant API as FastAPI
    participant CS as Chat Service
    participant LGS as LangGraph Service
    participant AM as Agent Manager
    participant PA as Planner Agent
    participant IVCA as Image/Video Creator
    participant TS as Tool Service
    participant SP as Stream Processor
    participant WS as WebSocket Service
    participant DB as Database
    
    Note over C,DB: User Request Initiation
    C->>API: POST /api/chat
    API->>CS: handle_chat(data)
    CS->>DB: create_chat_session()
    CS->>DB: create_message()
    
    Note over C,DB: LangGraph Agent Processing
    CS->>LGS: langgraph_multi_agent()
    LGS->>LGS: _fix_chat_history()
    LGS->>LGS: _create_text_model()
    LGS->>AM: create_agents()
    AM->>PA: create planner agent
    AM->>IVCA: create creator agent
    LGS->>LGS: create_swarm()
    
    Note over C,DB: Stream Processing
    LGS->>SP: process_stream()
    SP->>SP: compiled_swarm.astream()
    
    loop Stream Chunks
        SP->>SP: _handle_chunk()
        alt Message Chunk
            SP->>WS: send_to_websocket(delta)
            WS->>C: stream text content
        else Tool Call Chunk
            SP->>WS: send_to_websocket(tool_call)
            WS->>C: tool execution start
        else Values Chunk
            SP->>WS: send_to_websocket(all_messages)
            WS->>C: complete message update
            SP->>DB: create_message()
        end
    end
    
    Note over C,DB: Agent Reasoning & Tool Execution
    PA->>PA: analyze_user_request()
    PA->>TS: execute write_plan tool
    TS->>PA: plan created
    PA->>IVCA: transfer_to_image_video_creator()
    
    IVCA->>IVCA: create_design_strategy()
    IVCA->>TS: execute generation tool
    TS->>TS: route_to_provider()
    TS->>IVCA: generation result
    
    Note over C,DB: Completion
    SP->>WS: send_to_websocket(done)
    WS->>C: stream complete
    CS->>CS: remove_stream_task()
    
    Note over C,DB: Error Handling (if needed)
    alt Error Occurs
        LGS->>WS: send_to_websocket(error)
        WS->>C: error notification
    end
```

## 3. Agent Reasoning Flow

```mermaid
graph TD
    subgraph "User Input Processing"
        UI[User Input]
        LA[Language Analysis]
        TC[Task Classification]
    end
    
    subgraph "Planner Agent Reasoning"
        direction TB
        PA_START[Planner Agent Activated]
        PA_ANALYZE[Analyze Request Complexity]
        PA_DECIDE{Decision Point}
        PA_SIMPLE[Simple Task Execution]
        PA_COMPLEX[Complex Plan Creation]
        PA_PLAN[Generate Step-by-Step Plan]
        PA_QUANTITY[Parse Quantity Requirements]
        PA_HANDOFF[Transfer to Creator Agent]
    end
    
    subgraph "Image/Video Creator Reasoning"
        direction TB
        IVCA_START[Creator Agent Activated]
        IVCA_INPUT[Input Analysis]
        IVCA_DETECT{Input Type Detection}
        IVCA_STRATEGY[Create Design Strategy]
        IVCA_TOOL_SELECT[Select Optimal Tool]
        IVCA_BATCH{Batch Processing?}
        IVCA_EXECUTE[Execute Generation]
        IVCA_ERROR{Error Handling}
        IVCA_RETRY[Retry with Different Tool]
        IVCA_SUCCESS[Return Result]
    end
    
    subgraph "Tool Selection Logic"
        direction LR
        TOOL_TYPE{Task Type}
        IMG_TOOLS[Image Generation Tools]
        VID_TOOLS[Video Generation Tools]
        SYS_TOOLS[System Tools]
        
        subgraph "Image Tool Selection"
            IMG_INPUT{Input Images?}
            IMG_MULTI[Multiple Input Support]
            IMG_SINGLE[Single Input Support]
            IMG_TEXT[Text-to-Image Tools]
        end
        
        subgraph "Video Tool Selection"
            VID_INPUT{Input Type}
            VID_T2V[Text-to-Video]
            VID_I2V[Image-to-Video]
        end
    end
    
    %% Flow connections
    UI --> LA
    LA --> TC
    TC --> PA_START
    
    %% Planner reasoning
    PA_START --> PA_ANALYZE
    PA_ANALYZE --> PA_DECIDE
    PA_DECIDE -->|Simple| PA_SIMPLE
    PA_DECIDE -->|Complex| PA_COMPLEX
    PA_COMPLEX --> PA_PLAN
    PA_PLAN --> PA_QUANTITY
    PA_QUANTITY --> PA_HANDOFF
    PA_SIMPLE --> PA_HANDOFF
    PA_HANDOFF --> IVCA_START
    
    %% Creator reasoning
    IVCA_START --> IVCA_INPUT
    IVCA_INPUT --> IVCA_DETECT
    IVCA_DETECT --> IVCA_STRATEGY
    IVCA_STRATEGY --> IVCA_TOOL_SELECT
    IVCA_TOOL_SELECT --> IVCA_BATCH
    IVCA_BATCH -->|Yes| IVCA_EXECUTE
    IVCA_BATCH -->|No| IVCA_EXECUTE
    IVCA_EXECUTE --> IVCA_ERROR
    IVCA_ERROR -->|Success| IVCA_SUCCESS
    IVCA_ERROR -->|Failure| IVCA_RETRY
    IVCA_RETRY --> IVCA_TOOL_SELECT
    
    %% Tool selection
    IVCA_TOOL_SELECT --> TOOL_TYPE
    TOOL_TYPE -->|Image| IMG_TOOLS
    TOOL_TYPE -->|Video| VID_TOOLS
    TOOL_TYPE -->|System| SYS_TOOLS
    
    IMG_TOOLS --> IMG_INPUT
    IMG_INPUT -->|Yes, Multiple| IMG_MULTI
    IMG_INPUT -->|Yes, Single| IMG_SINGLE
    IMG_INPUT -->|No| IMG_TEXT
    
    VID_TOOLS --> VID_INPUT
    VID_INPUT -->|Text| VID_T2V
    VID_INPUT -->|Image| VID_I2V
    
    style PA_START fill:#f3e5f5
    style IVCA_START fill:#e8f5e8
    style PA_DECIDE fill:#ffeb3b
    style IVCA_DETECT fill:#ffeb3b
    style TOOL_TYPE fill:#ffeb3b
```

## 4. WebSocket Communication Pattern

```mermaid
sequenceDiagram
    participant C as Client
    participant WS as WebSocket
    participant SP as Stream Processor
    participant AG as Agents
    participant T as Tools
    
    Note over C,T: Connection Establishment
    C->>WS: Connect
    WS->>C: connected event
    
    Note over C,T: Chat Stream Processing
    C->>WS: User message (via HTTP)
    
    loop Stream Processing
        AG->>SP: AI message chunk
        alt Text Content
            SP->>WS: delta event
            WS->>C: {type: 'delta', text: '...'}
        else Tool Call Start
            SP->>WS: tool_call event
            WS->>C: {type: 'tool_call', id: '...', name: '...'}
        else Tool Arguments Stream
            SP->>WS: tool_call_arguments event
            WS->>C: {type: 'tool_call_arguments', id: '...', text: '...'}
        else Tool Execution Result
            T->>SP: Tool result
            SP->>WS: tool_call_result event
            WS->>C: {type: 'tool_call_result', id: '...', message: {...}}
        else Complete State Update
            SP->>WS: all_messages event
            WS->>C: {type: 'all_messages', messages: [...]}
        end
    end
    
    Note over C,T: Stream Completion
    SP->>WS: done event
    WS->>C: {type: 'done'}
    
    Note over C,T: Error Handling
    alt Error Occurs
        AG->>SP: Error
        SP->>WS: error event
        WS->>C: {type: 'error', error: '...'}
    end
    
    Note over C,T: Cancellation
    C->>WS: Cancel request (via HTTP)
    WS->>SP: Cancel stream
    SP->>C: Stream cancelled
```

## 5. Tool Architecture and Provider Routing

```mermaid
graph TB
    subgraph "Tool Service Layer"
        TS[Tool Service]
        TR[Tool Registry]
        TM[Tool Mapping]
    end
    
    subgraph "Tool Categories"
        direction LR
        subgraph "System Tools"
            WP[write_plan]
            CF[comfyui_dynamic]
        end
        
        subgraph "Image Tools"
            GPT[gpt_image_1_jaaz]
            IMG[imagen_4_jaaz]
            FLUX[flux_kontext_pro]
            RC[recraft_v3]
            IDG[ideogram3_bal]
            MJ[midjourney]
        end
        
        subgraph "Video Tools"
            SD[seedance_v1]
            KL[kling_v2]
            HL[hailuo_02]
            VEO[veo3_fast]
        end
    end
    
    subgraph "Provider Routing"
        direction TB
        PR[Provider Router]
        
        subgraph "External Providers"
            JAAZ[JAAZ API<br/>- GPT Image 1<br/>- Imagen 4<br/>- Seedance<br/>- Kling<br/>- Hailuo]
            REP[Replicate API<br/>- Flux Series<br/>- Recraft v3<br/>- Imagen 4]
            VOLCES[Volces API<br/>- Doubao Seedream<br/>- Seedance Pro/Lite]
            LOCAL[Local Providers<br/>- ComfyUI<br/>- Ollama]
        end
    end
    
    subgraph "Tool Execution Flow"
        direction TB
        TE[Tool Execution]
        PA[Parameter Analysis]
        PC[Provider Choice]
        ER[Error Recovery]
        FR[Fallback Routing]
    end
    
    %% Service connections
    TS --> TR
    TR --> TM
    
    %% Tool categorization
    TS --> WP
    TS --> CF
    TS --> GPT
    TS --> IMG
    TS --> FLUX
    TS --> RC
    TS --> IDG
    TS --> MJ
    TS --> SD
    TS --> KL
    TS --> HL
    TS --> VEO
    
    %% Provider routing
    GPT --> PR
    IMG --> PR
    FLUX --> PR
    RC --> PR
    SD --> PR
    KL --> PR
    HL --> PR
    VEO --> PR
    
    PR --> JAAZ
    PR --> REP
    PR --> VOLCES
    PR --> LOCAL
    
    %% Execution flow
    TS --> TE
    TE --> PA
    PA --> PC
    PC --> ER
    ER --> FR
    FR --> PC
    
    style TS fill:#e1f5fe
    style PR fill:#f3e5f5
    style TE fill:#e8f5e8
```

## 6. Database Schema and Data Flow

```mermaid
erDiagram
    CANVASES {
        string id PK
        string name
        datetime created_at
        datetime updated_at
    }
    
    CHAT_SESSIONS {
        string id PK
        string canvas_id FK
        string model
        string provider
        string title
        datetime created_at
        datetime updated_at
    }
    
    MESSAGES {
        integer id PK
        string session_id FK
        string role
        text content
        datetime created_at
    }
    
    DB_VERSION {
        integer version PK
    }
    
    CONFIG_SETTINGS {
        string key PK
        text value
        datetime updated_at
    }
    
    CANVASES ||--o{ CHAT_SESSIONS : contains
    CHAT_SESSIONS ||--o{ MESSAGES : contains
    
    %% Data flow annotations
    CANVASES ||--|| "Canvas Management" : manages
    CHAT_SESSIONS ||--|| "Session State" : tracks
    MESSAGES ||--|| "Conversation History" : stores
    CONFIG_SETTINGS ||--|| "System Configuration" : configures
```

This comprehensive set of mermaid diagrams illustrates the complete LangGraph architecture, from high-level system overview down to specific execution flows and data relationships. The architecture demonstrates sophisticated multi-agent orchestration with real-time streaming, robust error handling, and extensible tool management.