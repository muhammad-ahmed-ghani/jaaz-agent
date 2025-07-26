# LangGraph Architecture Analysis for `/server`

## Executive Summary

This repository implements a sophisticated multi-agent LangGraph architecture for generating images and videos through conversational AI. The system uses a swarm-based approach with specialized agents orchestrated through LangGraph's reactive agent framework.

## 1. Overall Architecture Overview

### Core Components
- **FastAPI Server**: HTTP/WebSocket API layer
- **LangGraph Multi-Agent System**: Core orchestration engine
- **Tool Service**: Extensible tool management system
- **WebSocket Communication**: Real-time streaming
- **Database Service**: SQLite-based persistence
- **Configuration Management**: TOML-based config system

### Technology Stack
- **LangGraph**: v0.4.8 (multi-agent orchestration)
- **LangChain**: Core LLM abstractions
- **FastAPI**: Web framework
- **Socket.IO**: Real-time communication
- **SQLite**: Data persistence
- **Anthropic/OpenAI**: LLM providers

## 2. LangGraph Multi-Agent Architecture

### Agent Types

#### 2.1 Planner Agent (`planner`)
**Purpose**: Task decomposition and execution planning
- **System Prompt**: Analyzes user requests and creates structured execution plans
- **Tools**: `write_plan` (system tool)
- **Handoffs**: Can transfer to `image_video_creator`
- **Key Behaviors**:
  - Breaks down complex tasks into steps
  - Handles quantity specifications (e.g., "20 images")
  - Maintains user language consistency
  - Automatically transfers image/video tasks

#### 2.2 Image Video Creator Agent (`image_video_creator`)
**Purpose**: Content generation specialist
- **System Prompt**: Professional image/video generation with design strategy
- **Tools**: Dynamic tool list based on configured providers
- **Handoffs**: Terminal agent (no further handoffs)
- **Key Behaviors**:
  - Creates design strategy documents
  - Handles batch generation (max 10 images per batch)
  - Supports input image processing
  - Comprehensive error handling
  - Supports both text-to-image and image-to-video workflows

### Agent Communication Pattern
```
User Request → Planner Agent → Planning → Transfer → Image/Video Creator → Tool Execution → Result
```

## 3. End-to-End Execution Flow

### 3.1 Request Initiation
1. **HTTP Request**: Client sends POST to `/api/chat`
2. **Data Extraction**: Parse messages, session_id, canvas_id, text_model, tool_list
3. **Session Management**: Create new chat session if first message
4. **Message Persistence**: Store user message in database

### 3.2 LangGraph Agent Processing
1. **Message History Fixing**: Remove incomplete tool calls
2. **Model Creation**: Instantiate text model (OpenAI/Ollama)
3. **Agent Creation**: Build agent instances with tools and handoffs
4. **Swarm Compilation**: Create multi-agent swarm with default active agent
5. **Context Setup**: Prepare canvas_id, session_id, tool_list context

### 3.3 Stream Processing
1. **Stream Initialization**: Start LangGraph astream with messages and context
2. **Chunk Processing**: Handle different chunk types:
   - `messages`: AI message chunks with content/tool calls
   - `values`: Complete state updates with all messages
   - `custom`: Custom events (if any)

### 3.4 Real-time Communication
1. **WebSocket Events**: Stream events to frontend:
   - `delta`: Text content streaming
   - `tool_call`: Tool execution start
   - `tool_call_arguments`: Tool parameter streaming
   - `tool_call_result`: Tool execution result
   - `all_messages`: Complete message history updates
   - `done`: Stream completion

### 3.5 Tool Execution
1. **Tool Selection**: Agent selects appropriate tool based on task
2. **Parameter Streaming**: Stream tool parameters in real-time
3. **Provider Routing**: Route to appropriate provider (jaaz, replicate, etc.)
4. **Result Processing**: Handle success/error cases
5. **Context Preservation**: Maintain canvas_id and session_id throughout

## 4. Agent Reasoning Flow

### 4.1 Planning Phase (Planner Agent)
```
User Input Analysis → Task Classification → Plan Generation → Tool Selection → Handoff Decision
```

**Decision Logic**:
- Simple tasks: Direct tool execution
- Complex tasks: Multi-step plan creation
- Image/Video tasks: Immediate transfer to creator agent
- Quantity awareness: Parse and preserve numerical requirements

### 4.2 Execution Phase (Creator Agent)
```
Task Reception → Design Strategy → Tool Selection → Batch Processing → Quality Assurance
```

**Reasoning Process**:
1. **Input Analysis**: Parse requirements and detect input images
2. **Strategy Formation**: Create professional design strategy document
3. **Tool Selection**: Choose optimal generation tool based on:
   - Task type (image vs video)
   - Input requirements (text vs image input)
   - Quality requirements
   - Batch size considerations
4. **Execution Planning**: Handle batch processing for large quantities
5. **Error Recovery**: Comprehensive error handling with user guidance

### 4.3 Tool Execution Reasoning
```
Context Analysis → Provider Selection → Parameter Optimization → Execution → Result Validation
```

## 5. Tool Architecture

### 5.1 Tool Management System
- **Tool Registry**: Centralized mapping in `tool_service.py`
- **Dynamic Loading**: Runtime tool discovery and initialization
- **Provider Abstraction**: Unified interface for multiple providers
- **Configuration-Driven**: Tool availability based on config

### 5.2 Tool Categories

#### Image Generation Tools
- **GPT Image 1**: Multi-input support, reference-based generation
- **Imagen 4**: Google's text-to-image model
- **Flux Series**: Kontext Pro/Max variants
- **Recraft v3**: Vector-style generation
- **Ideogram 3**: Balanced generation
- **Midjourney**: Professional artistic generation

#### Video Generation Tools
- **Seedance v1**: Text-to-video and image-to-video
- **Kling v2**: Advanced video generation
- **Hailuo 02**: Chinese video generation
- **Veo3 Fast**: Rapid video generation

#### System Tools
- **write_plan**: Planning and task decomposition
- **ComfyUI Integration**: Custom workflow execution

### 5.3 Tool Execution Pattern
```python
@tool("tool_name", description="...", args_schema=Schema)
async def tool_function(
    prompt: str,
    config: RunnableConfig,
    tool_call_id: Annotated[str, InjectedToolCallId],
    **kwargs
) -> str:
    # Extract context
    ctx = config.get('configurable', {})
    canvas_id = ctx.get('canvas_id', '')
    session_id = ctx.get('session_id', '')
    
    # Execute with provider
    return await provider_function(
        canvas_id=canvas_id,
        session_id=session_id,
        **parameters
    )
```

## 6. Data Flow Architecture

### 6.1 Message Flow
```
Client → FastAPI → Chat Service → LangGraph Agent → Tool Execution → WebSocket → Client
```

### 6.2 State Management
- **Session State**: Managed by database service
- **Agent State**: Maintained by LangGraph swarm
- **Tool State**: Context-aware execution state
- **WebSocket State**: Real-time connection management

### 6.3 Persistence Layer
- **Chat Sessions**: Canvas ID, model info, session metadata
- **Messages**: Role-based message history with JSON content
- **Configurations**: Provider settings, model configurations
- **Migration System**: Schema versioning and updates

## 7. Configuration and Extensibility

### 7.1 Provider Configuration
```toml
[jaaz]
url = "https://jaaz.app/api/v1/"
api_key = ""
max_tokens = 8192

[jaaz.models]
"gpt-4o" = { type = "text" }
"gpt-4o-mini" = { type = "text" }
```

### 7.2 Dynamic Tool Discovery
- **Tool Registration**: Automatic discovery from tool modules
- **Provider Integration**: Multi-provider support with failover
- **Custom Tools**: Extensible tool system for new capabilities

### 7.3 Agent Configuration
- **Modular Design**: Separate config classes for each agent
- **Handoff Management**: Declarative agent transfer definitions
- **Tool Assignment**: Dynamic tool allocation based on agent role

## 8. Error Handling and Resilience

### 8.1 Stream Error Handling
- **Cancellation Support**: Graceful task cancellation
- **Error Propagation**: WebSocket error notifications
- **Recovery Mechanisms**: Tool execution retry logic

### 8.2 Tool Error Handling
- **Provider Failover**: Automatic fallback to backup providers
- **Content Policy**: Sensitive content detection and guidance
- **API Error Recovery**: HTTP error handling with user feedback

### 8.3 Agent Error Handling
- **Message History Validation**: Automatic tool call cleanup
- **State Recovery**: Graceful degradation on agent failures
- **Context Preservation**: Maintain session state across errors

## 9. Performance and Scalability

### 9.1 Streaming Architecture
- **Real-time Processing**: WebSocket-based streaming
- **Chunk-based Processing**: Efficient message handling
- **Batch Optimization**: Intelligent batch processing for large requests

### 9.2 Resource Management
- **Task Lifecycle**: Proper task creation and cleanup
- **Connection Management**: WebSocket connection pooling
- **Memory Efficiency**: Stream-based processing to minimize memory usage

### 9.3 Scalability Considerations
- **Stateless Design**: Session state externalized to database
- **Provider Load Balancing**: Multiple provider support
- **Horizontal Scaling**: Database-backed session management

## 10. Security and Privacy

### 10.1 API Security
- **Input Validation**: Pydantic-based parameter validation
- **Content Filtering**: Provider-level content policy enforcement
- **Error Sanitization**: Safe error message handling

### 10.2 Data Privacy
- **Local Storage**: SQLite-based local data storage
- **Session Isolation**: Canvas and session-based data separation
- **Configuration Security**: Secure API key management

## 11. Development and Maintenance

### 11.1 Code Organization
- **Modular Architecture**: Clear separation of concerns
- **Type Safety**: Comprehensive type annotations
- **Documentation**: Inline documentation and docstrings

### 11.2 Testing and Debugging
- **Error Logging**: Comprehensive error tracking
- **Performance Monitoring**: WebSocket event tracking
- **Development Tools**: Debug-friendly logging and tracing

### 11.3 Extension Points
- **New Agents**: Easy agent addition through config system
- **New Tools**: Pluggable tool architecture
- **New Providers**: Provider abstraction for easy integration

This architecture demonstrates a sophisticated implementation of LangGraph's multi-agent capabilities, with strong emphasis on real-time user experience, extensibility, and robust error handling.