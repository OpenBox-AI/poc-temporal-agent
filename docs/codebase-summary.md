# Codebase Summary

**Generated:** 2026-01-31
**Version:** 0.2.0 (POC)
**Purpose:** Temporal AI Agent with OpenBox Governance - LLM-powered conversational agents in Temporal workflows

## Overview

Temporal AI Agent v0.2.0 demonstrates AI-driven workflows using Temporal orchestration with LiteLLM LLM integration, MCP (Model Context Protocol) support, and OpenBox governance. Users interact with configurable agent personas through a React UI, each with distinct tools and behaviors, executing tools only after explicit confirmation. Workflow state persists across 250+ turn conversations via continue-as-new pattern.

## Technology Stack

### Core Technologies
- **Python 3.10+** - Primary language
- **Temporal SDK** - Workflow orchestration (temporalio>=1.8.0)
- **FastAPI** - REST API backend (api/main.py)
- **React** - Frontend UI (frontend/)
- **LiteLLM** - LLM abstraction layer (litellm>=1.70.0)
- **OpenBox SDK** - Governance/observability (openbox-temporal-sdk-python)

### Infrastructure
- **OpenTelemetry** - HTTP/DB/file instrumentation
- **PostgreSQL** - Database operations (psycopg2-binary)
- **Docker** - Containerization (docker-compose.yml, Dockerfile)

## Project Structure

```
/Users/phuongvu/Code/openbox/samples/poc-ai-agent/
├── activities/              # Temporal activity implementations (584 LOC)
│   └── tool_activities.py   # LLM planning, validation, MCP, tool execution
├── api/                     # FastAPI REST server (221 LOC)
│   └── main.py             # 7 endpoints: send-prompt, confirm, end-chat, etc.
├── workflows/               # Temporal workflow definitions (639 LOC)
│   ├── agent_goal_workflow.py   # Main workflow (448 LOC)
│   └── workflow_helpers.py      # Helpers: timeouts, history, continue-as-new (191 LOC)
├── goals/                   # Agent goal definitions (680 LOC)
│   ├── __init__.py          # Goal aggregator, multi-goal mode
│   ├── agent_selection.py   # Agent selection + pirate treasure hunt
│   ├── travel.py           # Flight/train/event booking
│   ├── finance.py          # Account management, transfers
│   ├── hr.py              # PTO and paycheck goals
│   ├── food.py            # Food ordering with Stripe MCP
│   ├── ecommerce.py       # Order/package tracking
│   └── stripe_mcp.py      # Direct Stripe MCP access
├── tools/                   # Tool implementations (2000+ LOC)
│   ├── tool_registry.py    # Central ToolDefinition registry (474 LOC)
│   ├── change_goal.py      # Goal switching tool
│   ├── transfer_control.py # Control transfer tool
│   ├── give_hint.py        # Pirate treasure hunt hints
│   ├── search_flights.py   # Flight search (343 LOC)
│   ├── search_fixtures.py  # Premier League fixtures (389 LOC)
│   ├── find_events.py      # Oceania events (64 LOC)
│   ├── create_invoice.py   # Stripe invoice (79 LOC)
│   ├── list_agents.py      # Agent listing (43 LOC)
│   ├── ecommerce/          # Order (25), list (32), track (158)
│   ├── fin/                # Account (31), balances (33), transfer (151), loan (117)
│   ├── hr/                 # PTO helpers, booking
│   ├── food/               # Cart, Stripe setup
│   └── data/               # JSON fixtures
├── models/                  # Data types (141 LOC)
│   ├── data_types.py       # Workflow dataclasses (58 LOC)
│   └── tool_definitions.py # Tool/Agent/MCP definitions (83 LOC)
├── shared/                  # Shared utilities (257 LOC)
│   ├── config.py           # Temporal client config (62 LOC)
│   ├── mcp_client_manager.py   # MCP connection pooling (168 LOC)
│   └── mcp_config.py       # MCP server definitions (27 LOC)
├── prompts/                 # LLM prompt generation (282 LOC)
│   └── agent_prompt_generators.py
├── frontend/                # React 19 + Vite 6 + Tailwind 3 (1020 LOC)
│   ├── src/pages/App.jsx   # Main app (306 LOC)
│   ├── src/services/api.js # API service (136 LOC)
│   └── src/components/     # ChatWindow, MessageBubble, ConfirmInline, etc.
├── scripts/                 # Utility scripts (212 LOC)
│   ├── run_worker.py       # Worker with OpenBox (97 LOC)
│   ├── run_legacy_worker.py # Worker without OpenBox (32 LOC)
│   └── helpers
├── tests/                   # pytest test suite
│   ├── conftest.py          # Fixtures
│   ├── test_agent_goal_workflow.py
│   ├── test_mcp_integration.py
│   ├── test_tool_activities.py
│   └── test_workflow_helpers.py
├── enterprise/              # C# .NET TrainSearchWorker
│   ├── Activities/
│   └── Models/
├── thirdparty/              # Mock train booking server (216 LOC)
│   └── train_api.py
├── docs/                    # Documentation
├── .claude/                 # Claude Code configuration
│   ├── rules/              # Development rules
│   ├── skills/             # Skills
│   └── workflows/          # Workflows
├── .github/workflows/       # CI pipeline
│   └── ci.yml
├── .devcontainer/           # VS Code devcontainer
├── pyproject.toml          # Python dependencies
├── docker-compose.yml      # Services: PostgreSQL, Temporal
├── Dockerfile              # Python 3.10 slim
├── Makefile                # Build automation (1443 LOC)
└── README.md               # OpenBox SDK docs
```

**Total Python LOC:** ~5,100 core + tools + goals + scripts
**Total Frontend LOC:** ~1,020

## Core Components

### 1. Workflows (`workflows/`)

**`agent_goal_workflow.py`** - Main conversational agent workflow
- Manages conversation state (history, prompts, tool results)
- Handles user signals (prompt, confirm, end_chat)
- Validates user input against agent goals
- Executes tools via activities
- Supports goal switching and multi-goal mode
- Implements continue-as-new for long conversations (250 turns max)
- Dynamically loads MCP tools from server definitions

**Key Features:**
- Deterministic execution (HTTP calls via activities)
- User confirmation flow for tool execution
- Conversation history tracking
- Goal-based agent switching
- MCP server integration

### 2. Activities (`activities/`)

**`tool_activities.py`** - Non-deterministic operations (584 LOC)
- `agent_toolPlanner()` - LLM call to determine next action
- `agent_validatePrompt()` - Validate user input against goal
- `get_wf_env_vars()` - Lookup environment settings
- `mcp_list_tools()` - Load tools from MCP servers
- Domain-specific tool execution (finance, HR, e-commerce, food)
- MCP tool execution routing

**Characteristics:**
- OpenBox governance integration (activity interceptors)
- LiteLLM for multi-provider LLM access (OpenAI, Anthropic, Grok)
- MCP client lifecycle management via MCPClientManager
- Structured JSON output parsing with validation
- Idempotent activity design for retry safety

### 3. API (`api/main.py`)

**REST Endpoints:**
- `POST /send-prompt` - Send user message to workflow
- `POST /confirm` - Confirm tool execution
- `POST /end-chat` - End conversation
- `GET /get-conversation-history` - Query conversation state
- `GET /tool-data` - Get latest tool call details
- `GET /agent-goal` - Get current agent goal
- `POST /start-workflow` - Initialize new workflow

**Configuration:**
- CORS enabled for `localhost:5173` (frontend)
- Temporal client via `get_temporal_client()`
- Workflow ID: `agent-workflow` (singleton)

### 4. Goals (`goals/`)

Agent goals define:
- **Agent persona** (name, description)
- **Available tools** (from tool_registry)
- **Workflow description** (step-by-step execution plan)
- **Starter prompt** (initial agent message)
- **Example conversation** (few-shot learning)
- **MCP server definition** (optional external tools)

**Goal Categories:**
- `agent_selection` - Meta-agent for choosing agents
- `travel` - Flight/train booking, event finding
- `finance` - Account management, money transfers
- `hr` - PTO booking and management
- `food` - Restaurant ordering
- `ecommerce` - Order tracking, package tracking
- `stripe_mcp` - Stripe API via MCP

### 5. Tools (`tools/`)

**Tool Registry (`tool_registry.py`):**
- Defines tool schemas (name, description, arguments)
- Converts MCP tools to ToolDefinition format
- System tools: ListAgents, ChangeGoal
- Domain-specific tool collections

**Tool Categories:**
- System: Agent selection, goal switching
- Travel: SearchFlights, SearchTrains, FindEvents
- Finance: GetAccountBalances, MoveMoney, SubmitLoanApplication
- HR: BookPTO, CurrentPTO, FuturePTOCalc
- E-commerce: GetOrder, TrackPackage
- Food: AddToCart (with Stripe integration)

### 6. Shared Utilities (`shared/`)

**`config.py`** - Temporal client factory
- Supports local, mTLS, and API key auth
- Environment-driven configuration

**`mcp_client_manager.py`** - MCP server lifecycle
- Start/stop MCP servers (stdio, SSE)
- Connection pooling and cleanup
- Process management

**`mcp_config.py`** - MCP server configuration
- Load from `.mcp.json` or environment
- Server definition schema

### 7. Frontend (`frontend/`)

**React Components:**
- `ChatWindow.jsx` - Main conversation interface
- `MessageBubble.jsx` - User/agent message rendering
- `ConfirmInline.jsx` - Tool confirmation UI
- `LLMResponse.jsx` - Structured agent response display
- `NavBar.jsx` - Navigation header

**Features:**
- Real-time conversation updates (polling)
- Tool execution confirmation flow
- Conversation history persistence (localStorage)
- Error handling and loading states

## OpenBox Integration

### Governance Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Temporal Worker                          │
│                                                             │
│  ┌─────────────────┐          ┌─────────────────────────┐  │
│  │ Workflow        │          │ Activity                │  │
│  │ Interceptor     │          │ Interceptor             │  │
│  │                 │          │                         │  │
│  │ - Lifecycle     │          │ - Input/output capture │  │
│  │   events        │          │ - Guardrails            │  │
│  │ - Via activity  │          │ - HTTP spans            │  │
│  └─────────────────┘          └─────────────────────────┘  │
│         │                                 │                │
│         │                                 │                │
│         ▼                                 ▼                │
│  ┌───────────────────────────────────────────────────────┐ │
│  │         WorkflowSpanProcessor                         │ │
│  │  - Buffers spans per workflow_id                      │ │
│  │  - Merges HTTP body/header data                       │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                │
└───────────────────────────┼────────────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │   OpenBox Core           │
              │   POST /governance/      │
              │        evaluate          │
              │                          │
              │   Returns:               │
              │   - action: continue/stop│
              │   - guardrails_result    │
              └──────────────────────────┘
```

### Event Types

1. **WorkflowStarted** - Workflow begins execution
2. **WorkflowCompleted** - Workflow succeeds
3. **WorkflowFailed** - Workflow fails
4. **SignalReceived** - Signal received (user_prompt, confirm, etc.)
5. **ActivityStarted** - Activity begins (with input)
6. **ActivityCompleted** - Activity ends (with output + HTTP spans)

### Instrumentation

**HTTP Tracing:**
- Auto-instrumented: httpx, requests, urllib3
- Captures request/response bodies + headers
- Ignores OpenBox Core URLs (self-reference)

**Database Tracing:**
- PostgreSQL (psycopg2, asyncpg)
- MySQL, MongoDB, Redis
- Captures queries, not results

**File I/O Tracing:**
- Optional (disabled by default)
- Tracks open/read/write operations
- Skips system paths

**Custom Tracing:**
- `@traced` decorator for functions
- Manual span creation with `create_span()`

## Key Files

### Configuration Files

- **`pyproject.toml`** - Python 3.10+ dependencies, build config
- **`docker-compose.yml`** - PostgreSQL 14, Temporal Server
- **`Dockerfile`** - Python 3.10 slim, uvicorn
- **`.env`** - Environment variables (not in repo)
- **`.mcp.json`** - MCP server definitions (not in repo)
- **`.repomixignore`** - Codebase compaction exclusions
- **`Makefile`** - Build automation (1443 LOC)

### Entry Points

- **`scripts/run_worker.py`** (97 LOC) - Start Temporal worker with OpenBox
  - Uses `create_openbox_worker()` factory from SDK
  - Loads workflows and activities
  - Configures governance interceptors

- **`scripts/run_legacy_worker.py`** (32 LOC) - Worker without OpenBox

- **`api/main.py`** (221 LOC) - FastAPI application
  - 7 REST endpoints for workflow control
  - CORS for frontend (localhost:5173)
  - Temporal client via `get_temporal_client()`

### SDK Integration (v0.2.0)

OpenBox Temporal SDK **moved outside** this repo (commit 20a4623):
- **Location:** External `openbox-temporal-sdk-python` package
- **Import:** `from openbox import create_openbox_worker`
- **Factory:** Handles governance setup (interceptors, span collection, policy enforcement)

### Core Logic

- **`workflows/agent_goal_workflow.py`** (448 LOC) - Main workflow
  - Signal/query handlers (user_prompt, confirm, end_chat)
  - Conversation loop and state management
  - Tool execution flow with confirmation
  - Goal switching (ChangeGoal tool)
  - Continue-as-new at 250 turns

- **`activities/tool_activities.py`** (584 LOC) - Activity implementations
  - LLM integration via LiteLLM (multi-provider)
  - MCP client management and tool loading
  - Tool execution routing by name
  - Input validation and output parsing
  - Governance interceptor integration

- **`prompts/agent_prompt_generators.py`** (282 LOC) - Prompt engineering
  - Context generation for LLM
  - Tool descriptions from ToolDefinition objects
  - Multi-goal vs single-goal prompts
  - Few-shot example inclusion

## Development Workflows

### Running Locally

```bash
# Start Temporal server + PostgreSQL
docker-compose up temporal postgres

# Start worker with OpenBox
make run-worker
# OR
python scripts/run_worker.py

# Start API
make run-api
# OR
uvicorn api.main:app --reload

# Start frontend
cd frontend && npm run dev
```

### Running Tests

```bash
# All tests
make test

# Specific test
pytest tests/test_agent_goal_workflow.py
```

### Environment Variables

**Temporal Connection:**
- `TEMPORAL_ADDRESS` - Server address (default: localhost:7233)
- `TEMPORAL_NAMESPACE` - Namespace (default: default)
- `TEMPORAL_TASK_QUEUE` - Queue name (default: agent-task-queue)

**OpenBox Configuration:**
- `OPENBOX_URL` - OpenBox Core URL
- `OPENBOX_API_KEY` - API key for governance
- `OPENBOX_GOVERNANCE_POLICY` - fail_open / fail_closed

**Agent Configuration:**
- `AGENT_GOAL` - Initial goal (default: goal_event_flight_invoice)
- `SHOW_CONFIRM` - Require user confirmation for tools (default: true)
- `GOAL_CATEGORIES` - Comma-separated enabled categories

**LLM Configuration:**
- `LITELLM_MODEL` - Model name (e.g., gpt-4, claude-3-5-sonnet)
- `OPENAI_API_KEY` - OpenAI API key
- `ANTHROPIC_API_KEY` - Anthropic API key

## Dependencies

### Core Dependencies (pyproject.toml)
```toml
temporalio>=1.8.0          # Workflow engine
litellm>=1.70.0            # LLM abstraction
fastapi>=0.115.6           # REST API
uvicorn>=0.34.0            # ASGI server
pyyaml>=6.0.2              # Config parsing
python-dotenv>=1.0.1       # Environment loading
requests>=2.32.3           # HTTP client
httpx>=0.28.0              # Async HTTP client
pandas>=2.2.3              # Data manipulation
stripe>=11.4.1             # Payment processing
fastmcp>=2.7.0             # MCP protocol
psycopg2-binary>=2.9.10    # PostgreSQL driver
openbox-temporal-sdk-python # Governance SDK
```

### Dev Dependencies
```toml
pytest>=8.2                # Testing framework
pytest-asyncio>=0.26.0     # Async test support
black~=23.7                # Code formatter
isort~=5.12                # Import sorter
mypy>=1.16.0               # Type checker
poethepoet>=0.37.0         # Task runner
```

### Frontend Dependencies (package.json)
```json
{
  "react": "^18.x",
  "react-dom": "^18.x",
  "tailwindcss": "^3.x",
  "vite": "^5.x"
}
```

## Data Flow

### User Interaction Flow

1. **User sends message** → `POST /send-prompt`
2. **API signals workflow** → `user_prompt` signal
3. **Workflow validates input** → `agent_validatePrompt` activity
4. **Workflow calls LLM** → `agent_toolPlanner` activity
5. **LLM returns tool call** → `{next: "confirm", tool: "SearchFlights", args: {...}}`
6. **Frontend shows confirmation** → User clicks confirm
7. **User confirms** → `POST /confirm`
8. **Workflow executes tool** → Activity invocation
9. **Activity sends to OpenBox** → Governance evaluation
10. **Tool result returned** → Added to conversation history
11. **Frontend polls for updates** → `GET /get-conversation-history`

### Approval Flow (v0.2.0 - Polling)

Recent changes (commit 6376d4f) moved from Temporal signal-based approvals to polling from OpenBox Core:

1. **Activity starts** → ActivityGovernanceInterceptor intercepts
2. **Send ActivityStarted event** → OpenBox Core evaluates
3. **OpenBox returns guardrails** → Input redaction/validation rules
4. **Activity executes with redacted input** → HTTP calls captured as spans
5. **Activity completes** → Output captured and sent
6. **Workflow polls approval from Core** → Direct HTTP call (not Temporal signal)
7. **OpenBox returns action** → continue / stop / block
8. **Workflow proceeds or terminates** → Based on Core decision
9. **Configurable via env var:** `OPENBOX_URL`, `OPENBOX_API_KEY`, `OPENBOX_GOVERNANCE_POLICY`

## MCP Integration

### MCP Architecture

MCP (Model Context Protocol) enables dynamic tool loading from external servers:

1. **Server Definition** in goal:
```python
AgentGoal(
    id="goal_stripe_mcp",
    mcp_server_definition=MCPServerDefinition(
        name="stripe",
        command="uvx",
        args=["mcp-server-stripe"],
        env={"STRIPE_API_KEY": "sk_..."},
        included_tools=["create_payment_intent"]
    ),
    ...
)
```

2. **Workflow loads tools** → `load_mcp_tools()` activity
3. **MCP client starts server** → `mcp_client_manager.py`
4. **Server returns tool schemas** → JSON-RPC over stdio/SSE
5. **Tools added to goal** → Converted to ToolDefinition
6. **LLM sees tools** → Included in prompt context
7. **User triggers tool** → Activity calls MCP server
8. **Server executes** → Returns result
9. **Client stops server** → Cleanup on workflow end

### Supported MCP Servers

- **Stripe MCP** - Payment processing
- **Custom servers** - Any MCP-compliant server

## Testing

### Test Structure

```
tests/
├── test_agent_goal_workflow.py      # Workflow unit tests
├── test_tool_activities.py          # Activity unit tests
├── test_mcp_integration.py          # MCP integration tests
├── test_workflow_helpers.py         # Helper function tests
└── workflowtests/
    └── agent_goal_workflow_test.py  # Temporal workflow tests
```

### Testing Approaches

**Unit Tests:**
- Mock Temporal SDK components
- Test business logic in isolation
- Fast execution

**Integration Tests:**
- Use Temporal test environment
- Test workflow execution end-to-end
- Slower but comprehensive

**Manual Testing:**
- Start worker + API
- Interact via frontend
- Verify conversation flow

## Key Patterns

### 1. Agent Goal Pattern
- Goals define agent behavior
- Tools + prompt + examples = agent persona
- Goal switching for multi-agent conversations

### 2. Tool Confirmation Pattern
- LLM proposes tool call with args
- User confirms via UI
- Workflow executes after confirmation
- Configurable via `SHOW_CONFIRM`

### 3. Continue-as-New Pattern
- Long conversations (250+ turns)
- Workflow state reset
- History preserved in conversation_summary
- Prevents workflow size limits

### 4. MCP Dynamic Loading
- Tools loaded at workflow runtime
- No code changes for new tools
- Server lifecycle managed by workflow

### 5. OpenBox Governance Pattern
- Activity-level interception
- Input/output validation
- HTTP span collection
- Policy-based execution control

## Security Considerations

1. **API Key Management**
   - Store in `.env` (not committed)
   - Use environment variables
   - Rotate regularly

2. **Sensitive Data**
   - OpenBox guardrails redact PII
   - Database queries logged (no results)
   - HTTP bodies captured (sanitize sensitive endpoints)

3. **Workflow Isolation**
   - Single workflow ID (`agent-workflow`)
   - No cross-workflow data access
   - Temporal namespaces for multi-tenancy

4. **Input Validation**
   - `agent_validatePrompt` activity
   - Goal-based validation
   - Prevents prompt injection

## Performance Characteristics

**Workflow Execution:**
- Latency: 100-500ms per turn (LLM dependent)
- Throughput: 1 workflow per user session
- State size: 250 turns max (continue-as-new)

**API Response Times:**
- `/send-prompt`: ~50ms (signal)
- `/get-conversation-history`: ~10ms (query)
- `/tool-data`: ~10ms (query)

**Frontend:**
- Polling interval: 1s
- Initial load: ~200ms
- Message render: ~50ms

## Known Limitations

1. **Single Workflow Instance**
   - Workflow ID hardcoded as `agent-workflow`
   - Only one conversation per deployment
   - Multi-user requires workflow ID per user

2. **Tool Execution**
   - Sequential (no parallel tool calls)
   - No streaming responses
   - Confirmation required (or disabled globally)

3. **MCP Server Management**
   - Servers started per workflow
   - No connection pooling
   - Cleanup on workflow end

4. **Frontend State**
   - localStorage only (no backend persistence)
   - No conversation export
   - No search/filter

## Extension Points

**Adding New Agents:**
1. Create goal in `goals/` directory
2. Define tools in `tools/` directory
3. Implement tool activities in `activities/tool_activities.py`
4. Add to `goal_list` in `goals/__init__.py`

**Adding MCP Servers:**
1. Define server in `.mcp.json`
2. Create goal with `mcp_server_definition`
3. Specify `included_tools` (optional)

**Adding Custom Instrumentation:**
1. Use `@traced` decorator on functions
2. Create manual spans with `create_span()`
3. Add to OpenBox span collection

**Customizing Prompts:**
1. Edit `prompts/agent_prompt_generators.py`
2. Modify goal descriptions
3. Update example conversations

## References

- **Temporal Docs:** https://docs.temporal.io/
- **OpenBox SDK:** See project README.md
- **LiteLLM Docs:** https://docs.litellm.ai/
- **MCP Protocol:** https://modelcontextprotocol.io/
- **FastAPI Docs:** https://fastapi.tiangolo.com/
