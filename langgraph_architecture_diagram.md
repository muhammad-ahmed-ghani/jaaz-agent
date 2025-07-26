# LangGraph Architecture Diagrams

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Frontend"
        UI[React UI]
        WS[WebSocket Client]
    end
    
    subgraph "Backend Server"
        subgraph "API Layer"
            FastAPI[FastAPI Server]
            ChatRouter[Chat Router]
            WebSocketRouter[WebSocket Router]
        end
        
        subgraph "Service Layer"
            ChatService[Chat Service]
            WSService[WebSocket Service]
            ConfigService[Config Service]
            ToolService[Tool Service]
            DBService[Database Service]
        end
        
        subgraph "LangGraph Service"
            AgentService[Agent Service]
            AgentManager[Agent Manager]
            StreamProcessor[Stream Processor]
            
            subgraph "Agents"
                PlannerAgent[Planner Agent]
                CreatorAgent[Image/Video Creator Agent]
            end
        end
        
        subgraph "Tools Layer"
            SystemTools[System Tools<br/>- write_plan]
            ImageTools[Image Generation Tools<br/>- GPT Image<br/>- Flux<br/>- Midjourney<br/>- etc.]
            VideoTools[Video Generation Tools<br/>- Kling<br/>- Hailuo<br/>- Seedance<br/>- etc.]
        end
        
        DB[(SQLite Database)]
    end
    
    UI -->|HTTP Request| FastAPI
    UI <-->|WebSocket| WS
    WS <--> WebSocketRouter
    
    FastAPI --> ChatRouter
    ChatRouter --> ChatService
    
    ChatService --> AgentService
    ChatService --> DBService
    
    AgentService --> AgentManager
    AgentService --> StreamProcessor
    
    AgentManager --> PlannerAgent
    AgentManager --> CreatorAgent
    
    StreamProcessor --> WSService
    StreamProcessor --> DBService
    
    PlannerAgent --> SystemTools
    CreatorAgent --> ImageTools
    CreatorAgent --> VideoTools
    
    WSService --> WebSocketRouter
    DBService --> DB
    
    ToolService --> SystemTools
    ToolService --> ImageTools
    ToolService --> VideoTools
```

## 2. LangGraph Agent State Machine

```mermaid
stateDiagram-v2
    [*] --> Initialize: User Message
    
    Initialize --> PlannerAgent: Default Active Agent
    Initialize --> CreatorAgent: Previous Active Agent
    
    state PlannerAgent {
        [*] --> AnalyzeTask
        AnalyzeTask --> SimpleTask: No Planning Needed
        AnalyzeTask --> ComplexTask: Planning Required
        
        ComplexTask --> WritePlan: Execute write_plan tool
        WritePlan --> PlanComplete
        
        SimpleTask --> HandoffDecision
        PlanComplete --> HandoffDecision
        
        HandoffDecision --> [*]: Transfer to Creator
    }
    
    state CreatorAgent {
        [*] --> TaskAnalysis
        TaskAnalysis --> ImageGeneration: Image Task
        TaskAnalysis --> VideoGeneration: Video Task
        
        state ImageGeneration {
            [*] --> DesignStrategy
            DesignStrategy --> PromptCreation
            PromptCreation --> BatchCheck
            BatchCheck --> SingleGen: ≤10 images
            BatchCheck --> BatchGen: >10 images
            SingleGen --> ExecuteTool
            BatchGen --> ExecuteTool
            ExecuteTool --> [*]
        }
        
        state VideoGeneration {
            [*] --> VideoStrategy
            VideoStrategy --> DirectVideo: Text to Video
            VideoStrategy --> ImageFirst: Image to Video
            ImageFirst --> GenerateImages
            GenerateImages --> VideoFromImages
            DirectVideo --> ExecuteVideoTool
            VideoFromImages --> ExecuteVideoTool
            ExecuteVideoTool --> [*]
        }
    }
    
    PlannerAgent --> CreatorAgent: Handoff
    CreatorAgent --> StreamResults: Tool Execution
    StreamResults --> [*]: Task Complete
```

## 3. Message Flow and Stream Processing

```mermaid
sequenceDiagram
    participant User
    participant StreamProcessor
    participant Agent
    participant Tool
    participant WebSocket
    participant Database

    User->>Agent: Input Message
    
    Agent->>Agent: Process with LLM
    
    alt Text Response
        Agent->>StreamProcessor: AIMessageChunk(content)
        StreamProcessor->>WebSocket: {type: 'delta', text: content}
    else Tool Call
        Agent->>StreamProcessor: AIMessageChunk(tool_calls)
        StreamProcessor->>WebSocket: {type: 'tool_call', id, name}
        Agent->>Tool: Execute Tool
        
        loop Tool Execution
            Tool->>StreamProcessor: Streaming Arguments
            StreamProcessor->>WebSocket: {type: 'tool_call_arguments', text}
        end
        
        Tool-->>Agent: Tool Result
        Agent->>StreamProcessor: ToolMessage(result)
        StreamProcessor->>WebSocket: {type: 'tool_call_result', message}
    end
    
    Agent->>StreamProcessor: Values Chunk (complete state)
    StreamProcessor->>Database: Save Messages
    StreamProcessor->>WebSocket: {type: 'all_messages', messages}
    
    StreamProcessor->>WebSocket: {type: 'done'}
```

## 4. Tool Integration Architecture

```mermaid
graph LR
    subgraph "Tool Registration"
        ToolMapping[TOOL_MAPPING<br/>Dictionary]
        ToolInfo[Tool Info<br/>- ID<br/>- Display Name<br/>- Type<br/>- Provider<br/>- Function]
    end
    
    subgraph "Tool Service"
        GetTool[get_tool()]
        Initialize[initialize()]
        FilterTools[Filter by Type]
    end
    
    subgraph "Agent Configuration"
        PlannerTools[Planner Tools<br/>- write_plan]
        CreatorTools[Creator Tools<br/>- All Image Tools<br/>- All Video Tools]
    end
    
    subgraph "LangChain Integration"
        BaseTool[LangChain BaseTool]
        ToolWrapper[Tool Wrapper<br/>with Context]
    end
    
    ToolMapping --> ToolInfo
    ToolInfo --> GetTool
    
    Initialize --> FilterTools
    FilterTools --> PlannerTools
    FilterTools --> CreatorTools
    
    GetTool --> BaseTool
    BaseTool --> ToolWrapper
    
    PlannerTools --> Agent1[Planner Agent]
    CreatorTools --> Agent2[Creator Agent]
```

## 5. Handoff Mechanism

```mermaid
flowchart TB
    subgraph "Handoff Tool Creation"
        HandoffConfig[Handoff Configuration<br/>- agent_name<br/>- description]
        CreateHandoff[create_handoff_tool()]
        HandoffTool[Transfer Tool<br/>transfer_to_agent_name]
    end
    
    subgraph "Handoff Execution"
        AgentCall[Agent calls handoff tool]
        ToolMessage[Create ToolMessage<br/>'Successfully transferred']
        Command[Create Command<br/>- goto: target_agent<br/>- update: messages + active_agent]
    end
    
    subgraph "Swarm Processing"
        SwarmRouter[Swarm Router]
        TargetAgent[Target Agent Activation]
        ContinueExecution[Continue with new agent]
    end
    
    HandoffConfig --> CreateHandoff
    CreateHandoff --> HandoffTool
    
    HandoffTool --> AgentCall
    AgentCall --> ToolMessage
    ToolMessage --> Command
    
    Command --> SwarmRouter
    SwarmRouter --> TargetAgent
    TargetAgent --> ContinueExecution
```

## 6. Error Handling Flow

```mermaid
flowchart TD
    Start([Agent Execution]) --> Try{Try Block}
    
    Try -->|Success| Process[Process Normally]
    Try -->|Error| Catch[Catch Exception]
    
    Catch --> ErrorType{Error Type}
    
    ErrorType -->|Tool Error| ToolError[Tool Execution Failed]
    ErrorType -->|API Error| APIError[External API Failed]
    ErrorType -->|System Error| SystemError[Internal Error]
    
    ToolError --> UserGuidance[Provide User Guidance<br/>- Sensitive content warning<br/>- Alternative suggestions]
    APIError --> RetryAdvice[Suggest Retry<br/>- Check service status<br/>- Try different approach]
    SystemError --> LogError[Log Full Traceback]
    
    UserGuidance --> SendError[Send Error via WebSocket]
    RetryAdvice --> SendError
    LogError --> SendError
    
    SendError --> WSMessage[{type: 'error', error: message}]
    WSMessage --> Client([Client Notification])
    
    Process --> Complete([Task Complete])
```

## 7. Database Integration

```mermaid
erDiagram
    CHAT_SESSION {
        string session_id PK
        string model
        string provider
        string canvas_id
        string title
        datetime created_at
    }
    
    MESSAGE {
        int id PK
        string session_id FK
        string role
        json content
        datetime created_at
    }
    
    TOOL_EXECUTION {
        string tool_call_id PK
        string session_id FK
        string tool_name
        json arguments
        json result
        datetime executed_at
    }
    
    CHAT_SESSION ||--o{ MESSAGE : contains
    CHAT_SESSION ||--o{ TOOL_EXECUTION : includes
    MESSAGE ||--o| TOOL_EXECUTION : triggers
```

## 8. Configuration Flow

```mermaid
graph TD
    subgraph "Agent Configuration"
        BaseConfig[BaseAgentConfig<br/>- name<br/>- tools<br/>- system_prompt<br/>- handoffs]
        
        PlannerConfig[PlannerAgentConfig<br/>Extends BaseConfig]
        CreatorConfig[ImageVideoCreatorConfig<br/>Extends BaseConfig]
    end
    
    subgraph "Runtime Creation"
        AgentManager[AgentManager]
        CreateReactAgent[create_react_agent()]
        CompiledGraph[CompiledGraph]
    end
    
    subgraph "Swarm Assembly"
        AgentList[List of Agents]
        CreateSwarm[create_swarm()]
        ActiveSwarm[Active Swarm<br/>with default agent]
    end
    
    BaseConfig --> PlannerConfig
    BaseConfig --> CreatorConfig
    
    PlannerConfig --> AgentManager
    CreatorConfig --> AgentManager
    
    AgentManager --> CreateReactAgent
    CreateReactAgent --> CompiledGraph
    
    CompiledGraph --> AgentList
    AgentList --> CreateSwarm
    CreateSwarm --> ActiveSwarm
```