# Project Overview & Product Development Requirements

**Project:** Temporal AI Agent with OpenBox Governance (POC)
**Version:** 0.2.0
**Generated:** 2026-01-31
**Status:** Active Development

## Executive Summary

Temporal AI Agent is a proof-of-concept demonstrating LLM-powered conversational agents executing within Temporal workflows, with comprehensive governance and observability via OpenBox SDK. The system enables multi-turn conversations where AI agents interact with users, validate inputs, execute tools, and maintain conversation state—all under policy-based governance.

## Project Vision

Build a production-ready framework for conversational AI agents that:
- Execute tools safely within governed workflows
- Provide transparent observability of agent actions
- Support multiple agent personas/goals
- Enable dynamic tool loading via MCP protocol
- Maintain deterministic execution guarantees

## Core Value Propositions

### For Developers
- **Rapid Agent Development:** Define agents via goals (persona + tools + prompts)
- **MCP Integration:** Load external tools without code changes
- **Framework Flexibility:** Support any LLM provider via LiteLLM
- **Testing Infrastructure:** Temporal workflow replay testing

### For Operations
- **Governance:** OpenBox policies control agent behavior
- **Observability:** Complete HTTP/DB/file tracing
- **Reliability:** Temporal guarantees (retry, recovery, durability)
- **Security:** Input validation, PII redaction, policy enforcement

### For End Users
- **Natural Conversations:** Multi-turn dialogues with context
- **Tool Confirmation:** Explicit approval for sensitive actions
- **Agent Switching:** Change personas mid-conversation
- **Transparency:** See exactly what agents are doing

## Product Requirements

### Functional Requirements

#### FR-1: Conversational Agent Framework
- **Description:** Support multi-turn conversations with LLM-powered agents
- **Acceptance Criteria:**
  - Users send text prompts via API
  - Agents respond with natural language
  - Conversation history maintained across turns
  - Support 250+ turn conversations (via continue-as-new)
  - Agent responses include tool calls when appropriate

#### FR-2: Goal-Based Agent System
- **Description:** Multiple agent personas with different capabilities
- **Acceptance Criteria:**
  - Agents defined via AgentGoal objects (tools, prompts, examples)
  - Dynamic goal switching during conversations
  - Meta-agent for goal selection
  - Support travel, finance, HR, e-commerce, food ordering domains
  - Extensible to new domains

#### FR-3: Tool Execution with Confirmation
- **Description:** Agents execute tools after user approval
- **Acceptance Criteria:**
  - LLM proposes tool calls with arguments
  - Frontend displays confirmation UI
  - User explicitly approves/rejects
  - Tool execution blocked until confirmation
  - Configurable to skip confirmation (dev mode)

#### FR-4: MCP Protocol Integration
- **Description:** Load tools from external MCP servers
- **Acceptance Criteria:**
  - Goals specify MCP server definitions
  - Workflow dynamically loads tools from servers
  - Server lifecycle managed (start/stop)
  - Tools callable like native tools
  - Support stdio and SSE transports

#### FR-5: OpenBox Governance
- **Description:** Policy-based control of agent actions
- **Acceptance Criteria:**
  - Activity input/output captured
  - HTTP requests traced (body + headers)
  - Events sent to OpenBox Core
  - Policies can block workflows (stop action)
  - Guardrails redact sensitive data
  - Validation failures terminate workflows

#### FR-6: Input Validation
- **Description:** Validate user prompts against agent goals
- **Acceptance Criteria:**
  - Prompts validated before LLM call
  - Out-of-scope requests rejected
  - Clear error messages to users
  - Goal-specific validation rules
  - Prevent prompt injection

#### FR-7: REST API
- **Description:** HTTP API for frontend integration
- **Acceptance Criteria:**
  - Send prompts to workflow
  - Confirm tool execution
  - Query conversation history
  - Get current agent goal
  - End conversations
  - CORS configured for frontend

#### FR-8: React Frontend
- **Description:** Web UI for agent conversations
- **Acceptance Criteria:**
  - Chat interface with message bubbles
  - Tool confirmation buttons
  - Real-time updates (polling)
  - Conversation history display
  - Agent goal display
  - Responsive design

### Non-Functional Requirements

#### NFR-1: Reliability
- **Target:** 99.9% workflow completion rate
- **Implementation:**
  - Temporal retry policies
  - Idempotent activities
  - Error handling in all activities
  - Graceful degradation (fail-open mode)

#### NFR-2: Performance
- **Targets:**
  - API response time: <100ms (signals/queries)
  - LLM latency: <5s per turn
  - Workflow throughput: 100 concurrent conversations
  - Frontend render: <200ms
- **Implementation:**
  - Async activities
  - Connection pooling
  - Frontend memoization
  - Efficient state queries

#### NFR-3: Scalability
- **Targets:**
  - 1000+ concurrent workflows
  - 10K+ tools across MCP servers
  - Multi-tenant support
- **Implementation:**
  - Horizontal worker scaling
  - Stateless activities
  - Temporal namespace isolation
  - MCP server pooling

#### NFR-4: Security
- **Requirements:**
  - API key authentication
  - TLS for Temporal connections
  - Input validation at all entry points
  - Secrets in environment variables
  - PII redaction via OpenBox guardrails
- **Implementation:**
  - Environment-based config
  - No secrets in code
  - Temporal mTLS support
  - OpenBox policy enforcement

#### NFR-5: Observability
- **Requirements:**
  - Complete audit trail of agent actions
  - HTTP request/response capture
  - Database query logging
  - Workflow execution history
  - Error tracking
- **Implementation:**
  - OpenBox span collection
  - OpenTelemetry instrumentation
  - Temporal workflow history
  - Structured logging

#### NFR-6: Maintainability
- **Requirements:**
  - Clear code organization
  - Type hints throughout
  - Comprehensive tests
  - Documentation for all components
- **Implementation:**
  - Modular architecture
  - Type checking (mypy)
  - pytest test suite
  - Inline documentation

#### NFR-7: Extensibility
- **Requirements:**
  - Add new agents without code changes
  - Support any LLM provider
  - Plug in external tools (MCP)
  - Custom governance policies
- **Implementation:**
  - Goal-based agent definition
  - LiteLLM abstraction
  - MCP protocol
  - OpenBox policy engine

## Technical Architecture

### System Components

```
┌──────────────────────────────────────────────────────────┐
│                     Frontend (React)                      │
│  - Chat UI                                               │
│  - Tool confirmation                                     │
│  - Conversation display                                  │
└──────────────────┬───────────────────────────────────────┘
                   │ HTTP
                   ▼
┌──────────────────────────────────────────────────────────┐
│                  FastAPI Backend                          │
│  - /send-prompt                                          │
│  - /confirm                                              │
│  - /get-conversation-history                            │
└──────────────────┬───────────────────────────────────────┘
                   │ Temporal Signals/Queries
                   ▼
┌──────────────────────────────────────────────────────────┐
│               Temporal Workflow Engine                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │        AgentGoalWorkflow                            │ │
│  │  - Conversation state                               │ │
│  │  - Signal handlers                                  │ │
│  │  - Query handlers                                   │ │
│  └────────────────────────────────────────────────────┘ │
│                       │                                  │
│                       ▼                                  │
│  ┌────────────────────────────────────────────────────┐ │
│  │          Activities                                 │ │
│  │  - agent_toolPlanner (LLM)                         │ │
│  │  - agent_validatePrompt (LLM)                      │ │
│  │  - Tool executions                                  │ │
│  │  - MCP client calls                                 │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ├─────► OpenBox Core (Governance)
                   ├─────► LiteLLM (LLM Providers)
                   ├─────► MCP Servers (External Tools)
                   └─────► PostgreSQL / External APIs
```

### Data Flow

**User Message Flow:**
1. User types message in frontend
2. Frontend POST to `/send-prompt`
3. API sends `user_prompt` signal to workflow
4. Workflow validates prompt via activity
5. Workflow calls LLM via activity (with governance)
6. LLM returns tool call or response
7. Workflow updates conversation history
8. Frontend polls `/get-conversation-history`
9. UI displays agent response
10. If tool call, show confirmation button
11. User clicks confirm
12. Frontend POST to `/confirm`
13. Workflow executes tool via activity (with governance)
14. Tool result added to history
15. Loop continues

**Governance Flow:**
1. Activity interceptor captures input
2. Send ActivityStarted to OpenBox
3. OpenBox evaluates policies
4. Returns guardrails (redaction/validation)
5. Activity executes with modified input
6. HTTP spans collected during execution
7. Activity interceptor captures output
8. Send ActivityCompleted to OpenBox (with spans)
9. OpenBox evaluates policies
10. Returns action (continue/stop)
11. Workflow proceeds or terminates

## Development Roadmap

### Phase 1: Core Framework (Complete)
- ✅ Temporal workflow implementation
- ✅ FastAPI backend (7 endpoints)
- ✅ React 19 frontend with Vite
- ✅ Goal-based multi-agent system (8 goal types)
- ✅ Tool execution with user confirmation flow
- ✅ OpenBox SDK integration (external package)

### Phase 2: Enhanced Governance (Complete)
- ✅ HTTP tracing (body + headers, request/response)
- ✅ Database query tracing (SQL capture)
- ✅ File I/O tracing (optional)
- ✅ Custom function tracing (@traced decorator)
- ✅ Guardrails (input/output redaction)
- ✅ Approval polling from Core (commit 6376d4f, replaces Temporal signals)
- ⏳ Policy management UI
- ⏳ Real-time policy updates

### Phase 3: MCP Ecosystem (Complete)
- ✅ MCP protocol client (fastmcp)
- ✅ Stdio transport (subprocess)
- ✅ Dynamic tool loading (mcp_list_tools activity)
- ✅ Stripe MCP server integration
- ✅ Connection pooling (MCPClientManager)
- ✅ Server lifecycle management
- ⏳ SSE transport

### Phase 4: Production Readiness (Planned)
- ⏳ Multi-user support (workflow ID per user)
- ⏳ Authentication/authorization
- ⏳ Conversation persistence (database)
- ⏳ Streaming responses
- ⏳ Parallel tool execution
- ⏳ Advanced error recovery

### Phase 5: Enterprise Features (Future)
- 🔜 Multi-tenancy
- 🔜 RBAC for agent access
- 🔜 Audit logging
- 🔜 Compliance reporting
- 🔜 Custom LLM fine-tuning
- 🔜 Agent analytics dashboard

## Success Metrics

### Product Metrics
- **Agent Response Quality:** 90%+ user satisfaction
- **Tool Execution Success Rate:** 95%+
- **Conversation Completion Rate:** 80%+
- **Average Turns per Conversation:** 10-15

### Technical Metrics
- **Workflow Success Rate:** 99%+
- **API Uptime:** 99.9%+
- **P95 Response Time:** <3s
- **Error Rate:** <1%

### Business Metrics
- **Agent Development Time:** <1 day per agent
- **MCP Server Onboarding:** <1 hour
- **Policy Deployment Time:** <5 minutes
- **Developer Productivity:** 50% faster than custom builds

## Risk Assessment

### Technical Risks

**Risk:** LLM hallucination leads to incorrect tool calls
- **Impact:** High (incorrect actions)
- **Likelihood:** Medium
- **Mitigation:** Input validation, tool confirmation, guardrails

**Risk:** Temporal workflow size limits
- **Impact:** Medium (conversation truncation)
- **Likelihood:** Low (continue-as-new implemented)
- **Mitigation:** 250 turn limit, conversation summary

**Risk:** OpenBox Core downtime
- **Impact:** High (governance disabled)
- **Likelihood:** Low
- **Mitigation:** Fail-open mode, health checks, redundancy

**Risk:** MCP server crashes
- **Impact:** Medium (tool unavailable)
- **Likelihood:** Medium
- **Mitigation:** Server health checks, fallback tools, retry logic

### Security Risks

**Risk:** Prompt injection attacks
- **Impact:** High (agent hijacking)
- **Likelihood:** Medium
- **Mitigation:** Input validation, prompt sanitization, guardrails

**Risk:** PII leakage in logs
- **Impact:** High (compliance violation)
- **Likelihood:** Medium
- **Mitigation:** OpenBox redaction, log filtering, encryption

**Risk:** Unauthorized tool execution
- **Impact:** High (data breach)
- **Likelihood:** Low
- **Mitigation:** Confirmation flow, OpenBox policies, RBAC

## Dependencies

### External Services
- **Temporal Cloud/Server:** Workflow orchestration
- **OpenBox Core:** Governance engine
- **LLM Providers:** OpenAI, Anthropic, etc.
- **MCP Servers:** External tool providers
- **PostgreSQL:** Data persistence

### Third-Party Libraries
- **temporalio:** Workflow SDK
- **litellm:** LLM abstraction
- **fastapi:** Web framework
- **opentelemetry:** Instrumentation
- **fastmcp:** MCP protocol

## Deployment Architecture

### Development
```
Local Machine:
├── Docker Compose (Temporal + PostgreSQL)
├── Python Worker (OpenBox enabled)
├── FastAPI Server
└── Vite Dev Server (React)
```

### Production
```
Cloud Infrastructure:
├── Temporal Cloud (Managed)
├── Kubernetes Cluster
│   ├── Worker Pods (auto-scaling)
│   ├── API Pods (load balanced)
│   └── Frontend (CDN)
├── PostgreSQL (Managed)
└── OpenBox Core (Managed/Self-hosted)
```

## Configuration Management

### Environment Variables
- **Temporal:** Address, namespace, task queue, auth
- **OpenBox:** URL, API key, policy mode
- **LLM:** Model, API keys
- **Agent:** Initial goal, categories, confirmation mode
- **MCP:** Server definitions

### Configuration Files
- **`.env`:** Secrets, environment-specific config
- **`.mcp.json`:** MCP server definitions
- **`pyproject.toml`:** Python dependencies
- **`docker-compose.yml`:** Local development services

## Testing Strategy

### Unit Tests
- Workflow logic (mocked Temporal SDK)
- Activity logic (mocked external services)
- Tool implementations
- Helper functions

### Integration Tests
- Temporal workflow execution
- API endpoints
- MCP client integration
- Database operations

### End-to-End Tests
- Full conversation flows
- Tool execution with confirmation
- Goal switching
- Error handling

### Governance Tests
- Policy enforcement
- Guardrails validation
- Span collection
- Fail-open/closed modes

## Documentation Requirements

### Developer Documentation
- Architecture overview ✅
- API reference ✅
- Workflow design ✅
- Adding new agents ✅
- MCP integration guide ✅

### Operations Documentation
- Deployment guide ⏳
- Configuration reference ✅
- Monitoring setup ⏳
- Troubleshooting guide ⏳

### User Documentation
- Getting started ✅
- Agent capabilities ✅
- Tool confirmation ✅
- FAQ ⏳

## Compliance & Governance

### Data Privacy
- PII redaction via guardrails
- User consent for data collection
- Right to deletion (conversation cleanup)
- Data retention policies

### Security Standards
- OWASP Top 10 mitigation
- API key rotation
- TLS everywhere
- Regular security audits

### Audit Trail
- Complete workflow history
- Governance decisions logged
- Tool execution records
- User action tracking

## Open Questions

1. **Multi-user support:** Workflow ID strategy (per-user vs per-session)?
2. **Conversation persistence:** Database schema design for history storage?
3. **Streaming responses:** How to integrate with Temporal's determinism?
4. **Parallel tool execution:** Workflow design for concurrent activities?
5. **Policy versioning:** How to handle policy updates mid-conversation?
6. **Agent handoff:** Protocol for transferring conversations between agents?
7. **Cost optimization:** LLM token usage tracking and limits?

## References

- **Temporal Documentation:** https://docs.temporal.io/
- **OpenBox SDK README:** `/Users/phuongvu/Code/openbox/poc-ai-agent/README.md`
- **MCP Protocol Spec:** https://modelcontextprotocol.io/
- **LiteLLM Docs:** https://docs.litellm.ai/
- **Project Repository:** (Internal)
