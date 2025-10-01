# Microsoft Agent Framework - Low-Level Design and Features

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Package Structure](#package-structure)
5. [Key Features](#key-features)
6. [Design Patterns](#design-patterns)
7. [Extensibility](#extensibility)
8. [Development Standards](#development-standards)
9. [References](#references)

---

## Overview

Microsoft Agent Framework is a comprehensive multi-language framework for building, orchestrating, and deploying AI agents with support for both .NET and Python implementations. The framework provides everything from simple chat agents to complex multi-agent workflows with graph-based orchestration.

### Design Philosophy

The framework is built on the following core principles:

1. **Developer Experience First**: Simple, intuitive APIs with flat import structure
2. **Multi-Language Support**: Consistent APIs across Python and C#/.NET
3. **Async by Default**: All operations are asynchronous for better performance
4. **Extensibility**: Modular architecture supporting custom connectors and integrations
5. **Type Safety**: Strong typing with Pydantic models (Python) and native types (.NET)
6. **Observability**: Built-in OpenTelemetry integration for distributed tracing

---

## Architecture

### High-Level Architecture

The framework follows a tiered architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│           (User code, samples, applications)                 │
└─────────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────────┐
│                    Tier 0: Core Framework                    │
│     Agents │ Tools │ Types │ Threads │ Logging │ Memory    │
└─────────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────────┐
│              Tier 1: Advanced Components                     │
│   Workflows │ Middleware │ Context Providers │ Telemetry    │
│   Vector Data │ Observability │ Exceptions                   │
└─────────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────────┐
│          Tier 2: Connectors & Integrations                   │
│   OpenAI │ Azure │ Anthropic │ Redis │ Mem0 │ DevUI │ Lab  │
└─────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
User Application
      │
      ├──> Agent (ChatAgent, ResponsesAgent, AssistantsAgent)
      │      │
      │      ├──> Chat Client (OpenAI, Azure, Anthropic, etc.)
      │      │      │
      │      │      └──> LLM Provider API
      │      │
      │      ├──> Tools (Functions, MCP, OpenAPI)
      │      │
      │      ├──> Middleware (Request/Response Pipeline)
      │      │
      │      └──> Memory/Context Providers
      │
      └──> Workflow (Graph-based Orchestration)
             │
             ├──> Executors (Agent, Function, Custom)
             │
             ├──> Edges (Sequential, Conditional, Parallel)
             │
             ├──> State Management (Shared State, Checkpoints)
             │
             └──> Event Streaming (Real-time Updates)
```

---

## Core Components

### 1. Agents

Agents are the primary interface for interacting with LLMs. The framework provides three types:

#### ChatAgent

Direct interaction with chat completion APIs. Manages conversation state and tool execution.

**Key Features:**
- Conversation history management
- Tool/function calling support
- Streaming responses
- Middleware support
- Context injection

**Implementation Details:**
- File: `python/packages/core/agent_framework/_agents.py`
- Async-first design
- Supports multiple chat clients via protocol interface
- Automatic message history tracking
- Tool execution with error handling

**Example Flow:**
```
User Input → Middleware Pipeline → Chat Client → LLM → Tool Execution → Response
```

#### ResponsesAgent

Wraps OpenAI/Azure Responses API with managed threads and state.

**Key Features:**
- Server-side conversation management
- Automatic thread persistence
- File attachments support
- Built-in code interpreter and file search

**Implementation Details:**
- File: `python/packages/core/agent_framework/openai/_responses_client.py`
- Server manages conversation state
- Reduces client-side complexity
- Integrated tool execution

#### AssistantsAgent

Wraps OpenAI/Azure Assistants API (legacy support).

**Key Features:**
- Compatible with existing Assistant API code
- Server-side thread management
- Similar to ResponsesAgent but using older API

### 2. Tools

Tools extend agent capabilities through function calling.

#### Function Tools (`@ai_function` decorator)

**Design:**
- Python functions decorated with `@ai_function`
- Automatic schema generation from type hints
- Annotated types for parameter descriptions
- Async and sync function support

**Implementation Details:**
- File: `python/packages/core/agent_framework/_tools.py`
- Uses Python's `inspect` module for introspection
- Generates JSON schema from function signatures
- Supports complex types via Pydantic models

**Example:**
```python
@ai_function(description="Get weather for a location")
async def get_weather(
    location: Annotated[str, "City name"],
    units: Literal["celsius", "fahrenheit"] = "celsius"
) -> str:
    # Implementation
    return f"Weather in {location}"
```

#### MCP (Model Context Protocol) Tools

**Design:**
- Integration with MCP servers
- Dynamic tool discovery
- Server-side tool execution

**Implementation Details:**
- File: `python/packages/core/agent_framework/_mcp.py`
- Connects to MCP servers via stdio/HTTP
- Automatic tool schema mapping
- Error handling and retries

#### OpenAPI Tools

**Design:**
- Generate tools from OpenAPI specifications
- Automatic parameter validation
- HTTP client integration

### 3. Workflows

Graph-based orchestration system for multi-agent and multi-step processes.

#### Architecture

```
Workflow
  ├── Executors (Nodes)
  │   ├── AgentExecutor
  │   ├── FunctionExecutor
  │   └── Custom Executors
  │
  ├── Edges (Connections)
  │   ├── Sequential
  │   ├── Conditional
  │   ├── Switch/Case
  │   └── Parallel (Fan-out/Fan-in)
  │
  ├── State Management
  │   ├── Shared State (Cross-executor)
  │   └── Checkpoints (Persistence)
  │
  └── Events
      ├── ExecutorInvokeEvent
      ├── ExecutorCompletedEvent
      ├── WorkflowInputEvent
      └── WorkflowOutputEvent
```

#### Key Concepts

**Executors** (Nodes):
- Base class: `Executor`
- Define handlers with `@handler` decorator
- Process messages and emit results
- Can be agents, functions, or custom logic

**Edges** (Connections):
- Define message flow between executors
- Support conditions for routing
- Enable parallel execution patterns
- Handle result aggregation

**State Management**:
- **Shared State**: Key-value store accessible across executors
- **Checkpoints**: Persistent snapshots for pause/resume
- **Context**: Request-scoped data (RunContext, WorkflowContext)

**Event Streaming**:
- Real-time updates during execution
- Subscribe to specific event types
- Debug and monitor workflows
- Implement human-in-the-loop patterns

#### Implementation Details

**Files:**
- `_workflow.py`: Core Workflow class (~52K lines)
- `_executor.py`: Executor base class (~61K lines)
- `_edge.py`: Edge definitions (~32K lines)
- `_runner.py`: Execution engine (~21K lines)
- `_checkpoint.py`: Checkpoint management (~8K lines)

**Design Patterns:**
- Message-passing architecture
- Event-driven execution
- State machine pattern
- Builder pattern for construction

#### Advanced Features

**Magentic Orchestration:**
- AI-powered task planning and distribution
- Dynamic executor selection
- Adaptive workflow execution
- File: `_magentic.py` (~92K lines)

**Human-in-the-Loop:**
- Pause workflow for human input
- Approval gates
- Interactive decision points
- Resume from checkpoints

**Visualization:**
- Export workflow graphs
- GraphViz integration
- PNG/SVG output
- File: `_viz.py` (~14K lines)

### 4. Middleware

Pipeline pattern for request/response processing.

#### Architecture

```
Request → Middleware 1 → Middleware 2 → ... → Handler → Response
   ↑                                                          │
   └──────────────────────────────────────────────────────────┘
```

#### Types

**Function-based Middleware:**
```python
async def logging_middleware(context, next):
    logger.info(f"Request: {context.request}")
    response = await next(context)
    logger.info(f"Response: {response}")
    return response
```

**Class-based Middleware:**
```python
class RetryMiddleware(Middleware):
    async def __call__(self, context, next):
        for attempt in range(3):
            try:
                return await next(context)
            except Exception as e:
                if attempt == 2:
                    raise
                await asyncio.sleep(2 ** attempt)
```

**Decorator-based Middleware:**
```python
@middleware
async def auth_middleware(context, next):
    # Authentication logic
    return await next(context)
```

#### Implementation Details

- File: `python/packages/core/agent_framework/_middleware.py`
- Supports both agent-level and run-level middleware
- Composable pipeline
- Error handling and short-circuiting
- Shared state access

#### Common Use Cases

1. **Logging and Telemetry**: Track request/response
2. **Authentication**: Validate credentials
3. **Rate Limiting**: Control API usage
4. **Caching**: Store/retrieve responses
5. **Error Handling**: Centralized exception management
6. **Content Filtering**: Guardrails implementation
7. **Retry Logic**: Automatic retry on failure

### 5. Memory and Context Providers

#### Memory System

**Thread-based Memory:**
- File: `python/packages/core/agent_framework/_threads.py`
- Conversation history storage
- Multiple backend support (in-memory, Redis, file-based)
- Automatic message persistence

**Context Providers:**
- Inject contextual information into prompts
- Examples: user profiles, knowledge bases, time/date
- Extensible provider interface

**Redis Integration:**
- Distributed conversation storage
- High-performance caching
- Session management
- Package: `agent_framework.redis`

**Mem0 Integration:**
- Long-term memory for agents
- Semantic memory storage
- Contextual recall
- Package: `agent_framework.mem0`

#### Implementation Details

**ChatMessageStore Protocol:**
```python
class ChatMessageStore(Protocol):
    async def save(self, thread_id: str, messages: list[ChatMessage]) -> None: ...
    async def load(self, thread_id: str) -> list[ChatMessage]: ...
    async def clear(self, thread_id: str) -> None: ...
```

### 6. Types and Serialization

#### Type System

**Core Types:**
- File: `python/packages/core/agent_framework/_types.py`
- `ChatMessage`: Conversation messages
- `ChatToolMode`: Tool invocation modes
- `ContentPart`: Multimodal content
- `FunctionCallInfo`: Tool execution details

**Pydantic Integration:**
- File: `python/packages/core/agent_framework/_pydantic.py`
- All models inherit from `BaseModel`
- Automatic validation
- JSON serialization/deserialization
- Schema generation for tools

**Design Principle: Attributes over Inheritance**

Prefer:
```python
ChatMessage(role="user", content="Hello")
ChatMessage(role="assistant", content="Hi")
```

Avoid:
```python
UserMessage(content="Hello")
AssistantMessage(content="Hi")
```

**Rationale:**
- Reduces class proliferation
- Simpler API surface
- Easier to extend
- More flexible composition

#### Serialization

**Features:**
- File: `python/packages/core/agent_framework/_serialization.py`
- Checkpoint serialization
- State persistence
- JSON-safe encoding
- Custom serializers for complex types

### 7. Logging and Telemetry

#### Logging System

**Design:**
- File: `python/packages/core/agent_framework/_logging.py`
- Centralized logger factory
- Namespace-based loggers
- Configurable log levels
- Structured logging support

**Usage:**
```python
from agent_framework import get_logger

# Main package logger
logger = get_logger()

# Subpackage logger
logger = get_logger('agent_framework.azure')
```

**Implementation:**
- Wraps Python's `logging` module
- Consistent formatting
- Integration with observability tools

#### Telemetry

**Design:**
- File: `python/packages/core/agent_framework/_telemetry.py`
- User agent tracking
- Usage statistics
- Version information
- Privacy-conscious

**OpenTelemetry Integration:**
- File: `python/packages/core/agent_framework/observability.py`
- Distributed tracing
- Span creation and context propagation
- Metric collection
- Integration with tracing backends (Jaeger, Zipkin, etc.)

**Features:**
- Automatic span creation for agent operations
- Workflow execution tracing
- Tool invocation tracking
- Error and exception recording
- Custom attributes and events

### 8. Chat Clients

Abstraction layer for different LLM providers.

#### Design Hierarchy

```
ChatClientProtocol (Interface)
    │
    ├── OpenAIChatClient
    ├── AzureOpenAIChatClient
    ├── AnthropicChatClient
    ├── AzureAIChatClient
    └── Custom implementations
```

#### Implementation Details

**Files:**
- Core: `python/packages/core/agent_framework/_clients.py`
- OpenAI: `python/packages/core/agent_framework/openai/_chat_client.py`
- Azure: `python/packages/core/agent_framework/azure/_chat_client.py`

**Protocol Interface:**
```python
class ChatClientProtocol(Protocol):
    async def create_completion(
        self,
        messages: list[ChatMessage],
        tools: list[Tool] | None = None,
        **kwargs
    ) -> ChatResponse: ...
```

**Features:**
- Provider-agnostic interface
- Automatic retry logic
- Error handling
- Streaming support
- Tool/function calling mapping

**Environment-based Configuration:**
- Load credentials from environment variables
- Support for `.env` files
- Per-provider settings
- Secure credential management

---

## Package Structure

### Python Package Organization

#### Directory Structure

```
python/packages/
├── core/
│   └── agent_framework/           # Main package
│       ├── __init__.py            # Tier 0 exports
│       ├── _agents.py             # Agent implementations
│       ├── _tools.py              # Tool definitions
│       ├── _types.py              # Core types
│       ├── _workflows/            # Workflow system
│       │   ├── __init__.py
│       │   ├── _workflow.py
│       │   ├── _executor.py
│       │   ├── _edge.py
│       │   └── ...
│       ├── openai/                # OpenAI connector
│       ├── azure/                 # Azure connector
│       ├── observability.py       # Tier 1 component
│       └── exceptions.py          # Tier 1 component
│
├── lab/                           # Experimental features
│   └── agent_framework_lab/
│       ├── gaia/                  # GAIA benchmark
│       ├── tau2/                  # TAU2 benchmark
│       └── lightning/             # RL training
│
├── devui/                         # Developer UI
│   └── agent_framework_devui/
│
└── integrations/                  # Third-party integrations
    ├── mem0/
    ├── redis/
    └── ...
```

#### Import Structure

**Tier 0 (Core)**: Import directly from `agent_framework`
```python
from agent_framework import ChatAgent, ai_function, ChatMessage
```

**Tier 1 (Components)**: Import from `agent_framework.<component>`
```python
from agent_framework.observability import configure_tracing
from agent_framework.exceptions import AgentError
```

**Tier 2 (Connectors)**: Import from `agent_framework.<vendor>`
```python
from agent_framework.openai import OpenAIChatClient
from agent_framework.azure import AzureOpenAIChatClient
```

#### Lazy Loading

**Design:**
- Subpackages use lazy loading
- Defer imports until first use
- Reduce startup time
- Avoid unnecessary dependencies

**Implementation:**
```python
# In __init__.py
def __getattr__(name: str):
    if name in _IMPORTS:
        return getattr(importlib.import_module(PACKAGE_NAME), name)
    raise AttributeError(f"Module has no attribute {name}")
```

### Namespace Packages

**Purpose:**
- Independent package lifecycle
- Separate versioning
- Optional dependencies
- Modular installation

**Structure:**
```python
# agent_framework/__init__.py (namespace)
__path__ = __import__("pkgutil").extend_path(__path__, __name__)
```

**Benefits:**
- Install only needed components
- Reduce dependency bloat
- Clear separation of concerns
- Community extensions support

### Subpackage Design

**Naming Convention:**
- Format: `agent-framework-<vendor>-<feature>`
- Examples: `agent-framework-google-gemini`, `agent-framework-redis`
- Simpler vendors: `agent-framework-mem0`

**Installation:**
```bash
# Via extra
pip install agent-framework[redis]

# Direct install
pip install agent-framework-redis
```

**Optional Dependencies:**
```toml
[project.optional-dependencies]
redis = ["agent-framework-redis==1.0.0"]
mem0 = ["agent-framework-mem0==1.0.0"]
```

---

## Key Features

### 1. Graph-Based Workflows

#### Overview

Workflows provide a powerful graph-based orchestration system for building complex multi-agent and multi-step processes.

#### Core Capabilities

**Sequential Execution:**
- Chain executors in order
- Pass outputs between steps
- Maintain conversation context

**Parallel Execution:**
- Fan-out to multiple executors
- Concurrent processing
- Aggregation of results

**Conditional Routing:**
- Dynamic path selection
- Switch/case patterns
- Multi-selection fan-out

**Loops and Cycles:**
- Iterative refinement
- Feedback loops
- Until condition met

#### State Management

**Shared State:**
- Global key-value store
- Accessible by all executors
- Type-safe access
- Atomic operations

**Checkpointing:**
- Save execution state
- Pause and resume
- Time-travel debugging
- Human-in-the-loop approval

**Context Propagation:**
- Request-scoped data
- Conversation history
- User information
- Environment variables

#### Event System

**Event Types:**
- `WorkflowInputEvent`: Workflow started
- `ExecutorInvokeEvent`: Executor called
- `ExecutorCompletedEvent`: Executor finished
- `WorkflowOutputEvent`: Workflow completed
- Custom events

**Streaming:**
- Real-time progress updates
- Subscribe to event types
- Filter by executor
- Debugging and monitoring

#### Visualization

**Features:**
- Export workflow graphs
- PNG/SVG formats
- Node and edge labels
- Execution flow visualization

**Usage:**
```python
workflow.visualize("workflow.png")
```

### 2. DevUI - Developer Interface

#### Overview

Interactive developer UI for agent development, testing, and debugging.

**Package:** `agent-framework-devui`

#### Features

**Interactive Testing:**
- Test agents in real-time
- View conversation history
- Inspect tool calls
- Debug errors

**Workflow Visualization:**
- Real-time execution tracking
- Visual workflow graphs
- Step-by-step debugging
- State inspection

**Development Tools:**
- Schema validation
- Payload inspection
- Performance metrics
- Log viewing

#### Architecture

**Components:**
- Web server (FastAPI/Flask)
- WebSocket for streaming
- React frontend
- Agent discovery

**Integration:**
```python
from agent_framework.devui import serve

serve(agents=[my_agent], port=8000)
```

### 3. Lab Modules - Experimental Features

#### Overview

`agent-framework-lab` contains experimental features and research prototypes.

**Package:** `agent-framework-lab`

#### Modules

**GAIA Benchmark:**
- General assistant task evaluation
- Question answering
- Multi-step reasoning
- Package: `agent_framework.lab.gaia`

**TAU2 Benchmark:**
- Customer support task evaluation
- Tool usage assessment
- Conversation quality metrics
- Package: `agent_framework.lab.tau2`

**Lightning (RL Training):**
- Reinforcement learning for agents
- Policy optimization
- Reward modeling
- In development
- Package: `agent_framework.lab.lightning`

#### Design Philosophy

**Categories:**
1. **Incubation**: Features that may join core
2. **Research**: Experimental prototypes
3. **Benchmarks**: Evaluation tools

**Stability:**
- No stability guarantees
- Breaking changes expected
- Deprecation possible
- Use for experimentation only

### 4. Multimodal Support

#### Content Types

**Text:**
- UTF-8 strings
- Markdown formatting
- Code blocks

**Images:**
- URL references
- Base64 encoded
- Multiple formats (PNG, JPEG, WebP)

**Files:**
- Document upload
- File references
- Attachment handling

**Audio/Video:**
- Media references
- Transcription support
- Provider-dependent

#### Implementation

**ContentPart System:**
```python
ChatMessage(
    role="user",
    content=[
        TextContentPart(text="What's in this image?"),
        ImageContentPart(url="https://example.com/image.png")
    ]
)
```

**Provider Support:**
- OpenAI: GPT-4V, GPT-4o
- Azure: OpenAI models with vision
- Anthropic: Claude 3
- Google: Gemini

### 5. Observability and Monitoring

#### OpenTelemetry Integration

**Tracing:**
- Automatic span creation
- Context propagation
- Distributed tracing
- Parent-child relationships

**Metrics:**
- Agent invocation count
- Tool execution time
- Token usage
- Error rates

**Logging:**
- Structured logs
- Correlation IDs
- Log levels
- Contextual information

#### Configuration

```python
from agent_framework.observability import configure_tracing

configure_tracing(
    service_name="my-agent-app",
    endpoint="http://jaeger:4318",
    enable_console=True
)
```

#### Backends

**Supported:**
- Jaeger
- Zipkin
- Azure Application Insights
- AWS X-Ray
- Google Cloud Trace
- Console (development)

### 6. Exception Handling

#### Exception Hierarchy

```
AgentFrameworkException
    ├── AgentError
    │   ├── AgentRunError
    │   └── AgentConfigurationError
    ├── ToolError
    │   ├── ToolExecutionError
    │   └── ToolValidationError
    ├── WorkflowError
    │   ├── WorkflowExecutionError
    │   └── WorkflowValidationError
    └── ClientError
        ├── APIError
        └── AuthenticationError
```

#### Error Handling Patterns

**Try-Except:**
```python
try:
    result = await agent.run("Hello")
except AgentError as e:
    logger.error(f"Agent error: {e}")
    # Handle error
```

**Middleware-based:**
```python
async def error_handler(context, next):
    try:
        return await next(context)
    except Exception as e:
        logger.exception("Error occurred")
        return ErrorResponse(error=str(e))
```

**Workflow-level:**
- Error executors
- Fallback paths
- Retry strategies
- Graceful degradation

### 7. Authentication and Security

#### Azure Entra ID Integration

**Features:**
- Managed identity support
- Azure CLI authentication
- Token caching
- Automatic refresh

**Implementation:**
```python
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import DefaultAzureCredential

client = AzureOpenAIChatClient(
    credential=DefaultAzureCredential()
)
```

#### API Key Management

**Environment Variables:**
- Secure storage
- Per-provider keys
- Override support

**Configuration Files:**
- `.env` file support
- Profile-based config
- Development vs. production

#### Content Filtering

**Guardrails:**
- Input validation
- Output filtering
- PII detection
- Content moderation

**Middleware Implementation:**
```python
async def content_filter(context, next):
    if contains_pii(context.request):
        raise SecurityError("PII detected")
    return await next(context)
```

---

## Design Patterns

### 1. Async-First Design

#### Principle

All I/O operations are asynchronous by default.

**Rationale:**
- Better concurrency
- Improved performance
- Scalability
- Non-blocking operations

#### Implementation

**Async Functions:**
```python
async def my_function():
    result = await async_operation()
    return result
```

**Sync Wrappers:**
```python
def my_function_sync():
    return asyncio.run(my_function())
```

**Convention:**
- Use `async def` for async functions
- Use `def` for sync functions
- Always document async behavior

### 2. Protocol-Based Interfaces

#### Principle

Use protocols (structural subtyping) instead of abstract base classes.

**Rationale:**
- Duck typing support
- Flexibility
- No inheritance required
- Better testability

#### Example

```python
from typing import Protocol

class ChatClientProtocol(Protocol):
    async def create_completion(
        self,
        messages: list[ChatMessage],
        **kwargs
    ) -> ChatResponse: ...
```

**Benefits:**
- Any class implementing the methods works
- No need to inherit
- Easier mocking in tests

### 3. Builder Pattern

#### Principle

Use builders for complex object construction.

**Usage:**
- Workflows
- Sequential orchestration
- Concurrent orchestration
- Magentic workflows

#### Example

```python
workflow = (
    Workflow()
    .add_executor(executor1)
    .add_executor(executor2)
    .add_edge(executor1.id, executor2.id)
    .build()
)
```

**Benefits:**
- Fluent API
- Step-by-step construction
- Validation at build time
- Clear intent

### 4. Middleware Pipeline

#### Principle

Chain of responsibility for request processing.

**Pattern:**
```
Request → MW1 → MW2 → MW3 → Handler → MW3 → MW2 → MW1 → Response
```

**Characteristics:**
- Order matters
- Short-circuit support
- Composable
- Reusable

### 5. Event-Driven Architecture

#### Principle

Components communicate via events.

**Application:**
- Workflows
- Streaming responses
- Progress tracking
- Debugging

**Benefits:**
- Loose coupling
- Extensibility
- Observability
- Testability

### 6. Dependency Injection

#### Principle

Pass dependencies explicitly.

**Example:**
```python
agent = ChatAgent(
    name="MyAgent",
    chat_client=my_client,  # Injected
    tools=[tool1, tool2],    # Injected
    middleware=[mw1, mw2]    # Injected
)
```

**Benefits:**
- Testability
- Flexibility
- Clear dependencies
- No hidden state

### 7. Immutability

#### Principle

Core types are immutable where practical.

**Implementation:**
- Pydantic models with `frozen=True`
- Return new instances on modification
- State changes via new objects

**Benefits:**
- Thread safety
- Predictable behavior
- Easier reasoning
- Cacheable

---

## Extensibility

### 1. Custom Chat Clients

#### Implementation

Implement `ChatClientProtocol`:

```python
class MyChatClient:
    async def create_completion(
        self,
        messages: list[ChatMessage],
        tools: list[Tool] | None = None,
        **kwargs
    ) -> ChatResponse:
        # Custom implementation
        return ChatResponse(...)
```

**Integration:**
```python
agent = ChatAgent(
    name="MyAgent",
    chat_client=MyChatClient()
)
```

### 2. Custom Tools

#### Function Tools

```python
@ai_function(description="My custom tool")
async def my_tool(param: str) -> str:
    # Implementation
    return result
```

#### Class-Based Tools

```python
class MyTool:
    def __init__(self):
        self.name = "my_tool"
        self.description = "My custom tool"
    
    async def execute(self, **kwargs) -> str:
        # Implementation
        return result
```

### 3. Custom Executors

#### Implementation

Extend `Executor` base class:

```python
class MyExecutor(Executor):
    def __init__(self, id: str):
        super().__init__(id=id)
    
    @handler
    async def process(
        self,
        message: InputMessage,
        ctx: WorkflowContext[OutputMessage]
    ) -> None:
        # Process message
        result = await self.do_work(message)
        await ctx.send_message(OutputMessage(data=result))
```

### 4. Custom Middleware

#### Function-Based

```python
async def my_middleware(context, next):
    # Pre-processing
    result = await next(context)
    # Post-processing
    return result
```

#### Class-Based

```python
class MyMiddleware(Middleware):
    def __init__(self, config):
        self.config = config
    
    async def __call__(self, context, next):
        # Implementation
        return await next(context)
```

### 5. Custom Context Providers

#### Implementation

```python
class MyContextProvider:
    async def get_context(
        self,
        user_id: str,
        **kwargs
    ) -> dict[str, Any]:
        # Fetch contextual data
        return {"key": "value"}
```

**Integration:**
```python
agent = ChatAgent(
    name="MyAgent",
    chat_client=client,
    context_providers=[MyContextProvider()]
)
```

### 6. Custom Connectors/Subpackages

#### Structure

```
agent-framework-myvendor/
├── agent_framework_myvendor/
│   ├── __init__.py
│   ├── _client.py
│   └── py.typed
├── tests/
├── pyproject.toml
└── README.md
```

#### Implementation

```python
# agent_framework_myvendor/_client.py
class MyVendorChatClient:
    async def create_completion(self, messages, **kwargs):
        # Implementation
        pass

# agent_framework_myvendor/__init__.py
from ._client import MyVendorChatClient

__all__ = ["MyVendorChatClient"]
```

#### Lazy Loading in Core

```python
# In core agent_framework/myvendor/__init__.py
import importlib

PACKAGE_NAME = "agent_framework_myvendor"

def __getattr__(name: str):
    if name in _IMPORTS:
        return getattr(importlib.import_module(PACKAGE_NAME), name)
    raise ModuleNotFoundError(f"Install: pip install agent-framework-myvendor")
```

---

## Development Standards

### 1. Code Style

#### Formatting

**Tool:** Ruff
- Line length: 120 characters
- Target Python: 3.10+
- Auto-formatting on save

#### Linting

**Tool:** Ruff
- Type checking with mypy
- Import sorting
- Unused code detection

#### Commands

```bash
# Format code
uv run poe format

# Lint code
uv run poe lint

# Type check
uv run poe type-check
```

### 2. Documentation

#### Docstring Style

**Google Style Docstrings:**

```python
def my_function(arg1: str, arg2: int = 0) -> bool:
    """Single line description ending with period.

    Optional longer description with more details about
    the function's behavior.

    Args:
        arg1: Description of arg1.
        arg2: Description of arg2.
            Multi-line descriptions indented 4 spaces.

    Returns:
        Description of return value.

    Raises:
        ValueError: Description of when this is raised.
    """
    ...
```

#### File Headers

All files must start with:
```python
# Copyright (c) Microsoft. All rights reserved.
```

#### Type Hints

- Always use type hints
- Use `typing` module for complex types
- Include return types
- Use `typing.Protocol` for interfaces

### 3. Testing

#### Test Structure

```
tests/
├── unit/               # Unit tests
├── integration/        # Integration tests
└── e2e/               # End-to-end tests
```

#### Test Conventions

**Naming:**
- Files: `test_<module>.py`
- Classes: `Test<Feature>`
- Methods: `test_<scenario>`

**Markers:**
```python
@pytest.mark.unit
@pytest.mark.integration
@pytest.mark.asyncio
```

#### Commands

```bash
# Run all tests
uv run poe test

# Run specific package tests
uv run poe --directory packages/core test

# Run with coverage
uv run poe test-coverage
```

### 4. Dependency Management

#### Tool

**uv** - Fast Python package installer and resolver

#### Commands

```bash
# Install dependencies
uv sync --dev

# Add dependency
uv add package-name

# Update dependencies
uv lock --upgrade

# Create virtual environment
uv venv
```

#### pyproject.toml

**Structure:**
```toml
[project]
name = "agent-framework"
dependencies = [
    "pydantic>=2.0",
    "openai>=1.0",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]
azure = ["azure-identity", "azure-ai-openai"]
```

### 5. Version Control

#### Branch Strategy

- `main`: Stable releases
- `develop`: Development branch
- `feature/*`: Feature branches
- `fix/*`: Bug fix branches

#### Commit Messages

**Format:**
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `refactor`: Code refactoring
- `test`: Tests
- `chore`: Maintenance

#### Pre-commit Hooks

```bash
# Install hooks
uv run poe pre-commit-install

# Run manually
uv run poe pre-commit-run
```

### 6. Release Process

#### Versioning

**Semantic Versioning:** `MAJOR.MINOR.PATCH`

- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes

#### Release Steps

1. Update version in `pyproject.toml`
2. Update CHANGELOG.md
3. Create release tag
4. Build packages
5. Publish to PyPI

#### Commands

```bash
# Build package
uv run poe build

# Publish (requires credentials)
uv run poe publish
```

### 7. Performance Guidelines

#### Async Best Practices

- Use `asyncio.gather()` for concurrent operations
- Avoid blocking calls in async functions
- Use `asyncio.create_task()` for fire-and-forget
- Handle timeouts with `asyncio.wait_for()`

#### Memory Management

- Clean up resources in `finally` blocks
- Use context managers
- Avoid circular references
- Monitor memory usage in long-running processes

#### Caching

- Cache expensive computations
- Use LRU cache for deterministic functions
- Implement TTL for time-sensitive data
- Consider Redis for distributed caching

### 8. Security Practices

#### Credentials

- Never commit secrets
- Use environment variables
- Support credential managers
- Implement token refresh

#### Input Validation

- Validate all user input
- Sanitize strings
- Limit payload sizes
- Implement rate limiting

#### Dependencies

- Regular security audits
- Update dependencies
- Pin versions in production
- Use vulnerability scanners

---

## References

### Documentation

- [Main README](../README.md)
- [Python Dev Setup](../python/DEV_SETUP.md)
- [Design Documents](./design/)
- [Architectural Decisions](./decisions/)
- [Microsoft Learn Docs](https://learn.microsoft.com/agent-framework/)

### Design Documents

- [Python Package Setup](./design/python-package-setup.md)

### Architectural Decision Records

- [0001: Agent Run Response](./decisions/0001-agent-run-response.md)
- [0002: Agent Tools](./decisions/0002-agent-tools.md)
- [0003: OpenTelemetry Instrumentation](./decisions/0003-agent-opentelemetry-instrumentation.md)
- [0004: Foundry SDK Extensions](./decisions/0004-foundry-sdk-extensions.md)
- [0005: Python Naming Conventions](./decisions/0005-python-naming-conventions.md)
- [0006: User Approval](./decisions/0006-userapproval.md)
- [0007: Python Subpackages](./decisions/0007-python-subpackages.md)
- [0007: Agent Filtering Middleware](./decisions/0007-agent-filtering-middleware.md)

### Sample Code

- [Python Samples](../python/samples/)
- [.NET Samples](../dotnet/samples/)
- [Workflow Samples](../workflow-samples/)

### External Resources

- [OpenAI API Documentation](https://platform.openai.com/docs)
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [OpenTelemetry](https://opentelemetry.io/)
- [GraphViz](https://graphviz.org/)

---

## Appendix

### Glossary

**Agent**: An AI-powered assistant that can understand and respond to user requests, optionally using tools.

**Tool**: A function or capability that an agent can invoke to perform specific tasks.

**Workflow**: A graph-based orchestration of executors and edges defining a multi-step process.

**Executor**: A node in a workflow that processes messages and emits results.

**Edge**: A connection between executors defining message flow and routing.

**Checkpoint**: A persistent snapshot of workflow state enabling pause/resume.

**Middleware**: A component in the request/response pipeline for cross-cutting concerns.

**Context Provider**: A component that injects contextual information into agent prompts.

**Chat Client**: An abstraction for communicating with LLM providers.

**Streaming**: Real-time delivery of responses or events as they are generated.

**Tier 0/1/2**: Package organization levels (core, advanced, connectors).

### Acronyms

- **AI**: Artificial Intelligence
- **API**: Application Programming Interface
- **LLM**: Large Language Model
- **MCP**: Model Context Protocol
- **HITL**: Human-In-The-Loop
- **ADR**: Architectural Decision Record
- **PII**: Personally Identifiable Information
- **TTL**: Time To Live
- **CRUD**: Create, Read, Update, Delete
- **REST**: Representational State Transfer
- **JSON**: JavaScript Object Notation
- **CLI**: Command Line Interface
- **SDK**: Software Development Kit
- **RL**: Reinforcement Learning

### FAQ

**Q: When should I use workflows vs simple agents?**
A: Use workflows for multi-step processes, conditional logic, parallel execution, or when you need checkpointing. Use simple agents for straightforward conversational interactions.

**Q: How do I choose between ChatAgent, ResponsesAgent, and AssistantsAgent?**
A: Use ChatAgent for maximum control and flexibility. Use ResponsesAgent when you want server-side conversation management. Use AssistantsAgent for compatibility with legacy code.

**Q: Can I mix different LLM providers in a workflow?**
A: Yes! Each agent in a workflow can use a different chat client/provider.

**Q: How do I handle rate limits?**
A: Implement retry middleware with exponential backoff, or use workflow checkpoints to pause and resume.

**Q: Is the framework thread-safe?**
A: Core types are immutable and safe to share. Agents and workflows should not be shared between concurrent requests without proper synchronization.

**Q: Can I use synchronous code with the framework?**
A: The framework is async-first. You can wrap async calls with `asyncio.run()` for sync usage, but performance will be better with async throughout.

**Q: How do I debug workflow execution?**
A: Use event streaming, visualization, and the DevUI. Enable detailed logging and tracing for production debugging.

**Q: What's the difference between agent-framework and agent-framework-core?**
A: They are the same package. `agent-framework-core` is an alias for `agent-framework`.

### Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines on:
- Setting up development environment
- Submitting issues
- Creating pull requests
- Code review process
- Community guidelines

---

**Document Version:** 1.0  
**Last Updated:** 2025-01-01  
**Maintained By:** Microsoft Agent Framework Team
