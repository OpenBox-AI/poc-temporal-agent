# Code Standards & Conventions

**Generated:** 2026-01-31
**Version:** 0.2.0
**Purpose:** Coding standards, patterns, and best practices for Temporal AI Agent project

## Language Standards

### Python Style Guide

**Base Standard:** PEP 8 with project-specific extensions

**Formatting:**
- Line length: 88 characters (Black default)
- Indentation: 4 spaces
- String quotes: Double quotes preferred
- Trailing commas: Always in multi-line structures

**Tools:**
```bash
# Format code
black .
isort .

# Type checking
mypy --check-untyped-defs --namespace-packages .

# Linting
ruff check .
```

**Configuration:**
```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ['py310']

[tool.isort]
profile = "black"

[tool.mypy]
python_version = "3.10"
ignore_missing_imports = true
check_untyped_defs = true
namespace_packages = true
```

### Type Hints

**Required everywhere:**
```python
# Good
def get_account_balance(user_id: str) -> float:
    return 100.0

async def execute_tool(
    tool_name: str,
    args: Dict[str, Any],
    context: WorkflowContext
) -> ToolResult:
    ...

# Bad - no type hints
def get_account_balance(user_id):
    return 100.0
```

**Complex types:**
```python
from typing import Dict, List, Optional, Union, TypedDict, Literal

# TypedDict for structured data
class ToolData(TypedDict, total=False):
    next: Literal["confirm", "question", "pick-new-goal", "done"]
    tool: str
    args: Dict[str, Any]
    response: str
    force_confirm: bool

# Optional for nullable values
def get_user(user_id: str) -> Optional[User]:
    ...

# Union for multiple types
def process_response(data: Union[str, Dict[str, Any]]) -> str:
    ...
```

### Naming Conventions

**Python:**
- Classes: `PascalCase` (e.g., `AgentGoalWorkflow`, `ToolDefinition`)
- Functions/methods: `snake_case` (e.g., `execute_tool`, `get_conversation_history`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_TURNS_BEFORE_CONTINUE`, `TEMPORAL_TASK_QUEUE`)
- Private: `_leading_underscore` (e.g., `_internal_helper`)
- Workflows: `PascalCase` class + `snake_case` methods
- Activities: `snake_case` (e.g., `agent_tool_planner`, `get_wf_env_vars`)

**Files:**
- Python modules: `snake_case.py` (e.g., `tool_activities.py`, `agent_goal_workflow.py`)
- Documentation: `kebab-case.md` (e.g., `code-standards.md`, `system-architecture.md`)

**Variables:**
```python
# Good
user_id = "123"
conversation_history = []
is_confirmed = False
tool_data = {}

# Bad
userId = "123"  # camelCase
ConversationHistory = []  # PascalCase for variable
isconfirmed = False  # no underscore
toolData = {}  # camelCase
```

## Project Structure Standards

### Directory Organization

```
project_root/
├── workflows/          # Temporal workflow definitions
│   └── *_workflow.py   # Workflow files end with _workflow.py
├── activities/         # Temporal activity implementations
│   └── *_activities.py # Activity files end with _activities.py
├── goals/              # Agent goal definitions
│   └── *.py           # Domain-specific goals (travel.py, finance.py)
├── tools/              # Tool implementations
│   ├── __init__.py
│   ├── tool_registry.py  # Tool definitions
│   └── domain/           # Domain-specific tools
├── models/             # Data models and types
│   ├── data_types.py
│   └── tool_definitions.py
├── shared/             # Shared utilities
│   ├── config.py
│   └── *.py
├── prompts/            # Prompt engineering
│   └── *_generators.py
├── api/                # REST API
│   └── main.py
├── frontend/           # React frontend
│   └── src/
├── scripts/            # Development scripts
│   └── *.py
├── tests/              # Test suite
│   ├── test_*.py       # Unit tests
│   └── workflowtests/  # Temporal workflow tests
└── docs/               # Documentation
```

### File Size Guidelines

- **Workflows:** <500 lines (split if larger)
- **Activities:** <800 lines (split by domain)
- **Tools:** <200 lines per tool file
- **Models:** <300 lines (split by concern)
- **Utilities:** <400 lines (modularize if larger)

### Import Order

```python
# 1. Standard library
import os
from datetime import timedelta
from typing import Dict, List, Optional

# 2. Third-party libraries
from temporalio import workflow
from temporalio.common import RetryPolicy
from fastapi import FastAPI, HTTPException

# 3. Local imports (absolute)
from models.data_types import CombinedInput, ToolPromptInput
from workflows.workflow_helpers import handle_tool_execution
from activities.tool_activities import ToolActivities

# 4. Local imports (relative) - avoid if possible
from . import helpers
```

## Temporal-Specific Standards

### Workflow Design

**Determinism rules:**
```python
@workflow.defn
class AgentGoalWorkflow:
    def __init__(self) -> None:
        # ✅ Instance variables for state
        self.conversation_history: ConversationHistory = {"messages": []}
        self.prompt_queue: Deque[str] = deque()
        self.confirmed: bool = False

    @workflow.run
    async def run(self, input: CombinedInput) -> str:
        # ❌ NO non-deterministic operations
        # current_time = datetime.now()  # WRONG
        # user_id = random.randint(1, 100)  # WRONG
        # response = requests.get("https://api.example.com")  # WRONG

        # ✅ Use activities for non-deterministic operations
        result = await workflow.execute_activity_method(
            ToolActivities.get_current_time,
            start_to_close_timeout=timedelta(seconds=10)
        )

        # ✅ Use workflow.wait_condition for blocking
        await workflow.wait_condition(
            lambda: bool(self.prompt_queue) or self.chat_ended
        )

        # ✅ Use workflow.logger for logging
        workflow.logger.info(f"Processing message: {prompt}")
```

**Signal handlers:**
```python
@workflow.signal
async def user_prompt(self, prompt: str) -> None:
    """Signal handler for receiving user prompts."""
    workflow.logger.info(f"Signal received: user_prompt")
    if self.chat_ended:
        workflow.logger.warning(f"Message dropped (chat closed): {prompt}")
        return
    self.prompt_queue.append(prompt)

# ❌ Don't call activities from signals
# ❌ Don't perform long operations
# ✅ Only modify workflow state
```

**Query handlers:**
```python
@workflow.query
def get_conversation_history(self) -> ConversationHistory:
    """Query handler to retrieve conversation history."""
    return self.conversation_history

# ✅ Queries are read-only
# ✅ Return current state immediately
# ❌ Don't modify state
# ❌ Don't call activities
```

### Activity Design

**Activity implementation:**
```python
class ToolActivities:
    @staticmethod
    @activity.defn(name="agent_toolPlanner")
    async def agent_toolPlanner(input: ToolPromptInput) -> Dict[str, Any]:
        """Call LLM to determine next action."""
        activity.logger.info(f"Planning next step for prompt: {input.prompt[:50]}...")

        try:
            # ✅ Idempotent operations
            # ✅ HTTP calls allowed
            # ✅ Database operations allowed
            response = await llm_client.call(
                prompt=input.prompt,
                context=input.context_instructions
            )
            return response
        except Exception as e:
            # ✅ Log errors
            activity.logger.error(f"LLM call failed: {e}")
            # ✅ Let Temporal handle retry
            raise
```

**Activity timeouts:**
```python
# Standard timeouts
LLM_ACTIVITY_START_TO_CLOSE_TIMEOUT = timedelta(seconds=60)
LLM_ACTIVITY_SCHEDULE_TO_CLOSE_TIMEOUT = timedelta(seconds=90)

# Tool execution timeouts
TOOL_ACTIVITY_START_TO_CLOSE_TIMEOUT = timedelta(seconds=30)

# Usage
result = await workflow.execute_activity_method(
    ToolActivities.agent_toolPlanner,
    args=[input],
    start_to_close_timeout=LLM_ACTIVITY_START_TO_CLOSE_TIMEOUT,
    schedule_to_close_timeout=LLM_ACTIVITY_SCHEDULE_TO_CLOSE_TIMEOUT,
    retry_policy=RetryPolicy(
        initial_interval=timedelta(seconds=5),
        backoff_coefficient=2.0,
        maximum_interval=timedelta(seconds=60),
        maximum_attempts=3
    )
)
```

### Continue-as-New Pattern

```python
async def continue_as_new_if_needed(
    conversation_history: ConversationHistory,
    prompt_queue: Deque[str],
    goal: AgentGoal,
    max_turns: int,
    add_message_fn: callable
) -> None:
    """Continue workflow as new if turn limit reached."""
    if len(conversation_history["messages"]) >= max_turns:
        workflow.logger.info(f"Turn limit ({max_turns}) reached, continuing as new")

        # ✅ Summarize conversation before reset
        summary = await get_conversation_summary(conversation_history)

        # ✅ Pass minimal state to new workflow
        new_input = CombinedInput(
            tool_params=AgentGoalWorkflowParams(
                conversation_summary=summary,
                prompt_queue=prompt_queue
            ),
            agent_goal=goal
        )

        # ✅ Continue as new
        workflow.continue_as_new(new_input)
```

## MCP Integration Standards

### Using MCP Servers in Goals

```python
from models.tool_definitions import MCPServerDefinition

# Define goal with MCP server
goal_stripe = AgentGoal(
    id="goal_stripe_mcp",
    mcp_server_definition=MCPServerDefinition(
        name="stripe",
        command="uvx",
        args=["mcp-server-stripe"],
        env={"STRIPE_API_KEY": os.getenv("STRIPE_API_KEY")},
        included_tools=["create_payment_intent"]
    ),
    # ... other goal config
)
```

**Workflow handles:**
1. Server startup (stdio/SSE transport)
2. Tool loading via `mcp_list_tools` activity
3. Tool execution via MCP JSON-RPC
4. Server cleanup on workflow end

## OpenBox Integration Standards

### Governance Patterns

**Worker setup with OpenBox:**
```python
from openbox import create_openbox_worker

# ✅ Use factory function for full integration
worker = create_openbox_worker(
    client=client,
    task_queue=TEMPORAL_TASK_QUEUE,
    workflows=[AgentGoalWorkflow],
    activities=[
        ToolActivities.agent_toolPlanner,
        ToolActivities.agent_validatePrompt,
        # ... other activities
    ],
    # OpenBox config
    openbox_url=os.getenv("OPENBOX_URL"),
    openbox_api_key=os.getenv("OPENBOX_API_KEY"),
    governance_timeout=30.0,
    governance_policy="fail_closed",  # or "fail_open"

    # Instrumentation
    instrument_databases=True,
    instrument_file_io=False,  # Only enable if needed
)
```

**Activity design for governance:**
```python
@activity.defn(name="execute_payment")
async def execute_payment(payment_data: PaymentInput) -> PaymentResult:
    """Execute payment with OpenBox governance."""
    # ✅ Activity interceptor automatically captures input
    # ✅ OpenBox validates/redacts input before execution

    try:
        # ✅ HTTP calls are automatically traced
        result = await payment_api.charge(
            amount=payment_data.amount,
            card_token=payment_data.card_token
        )

        # ✅ Activity interceptor captures output
        # ✅ OpenBox validates/redacts output
        return result

    except GuardrailsValidationFailed as e:
        # ❌ Guardrails failed - workflow will terminate
        activity.logger.error(f"Guardrails validation failed: {e}")
        raise

    except GovernanceStop as e:
        # ❌ Policy blocked execution
        activity.logger.error(f"Governance stopped execution: {e}")
        raise
```

**Custom tracing:**
```python
from openbox.tracing import traced, create_span

# ✅ Trace important functions
@traced(capture_args=True, capture_result=True)
async def calculate_fraud_score(transaction: Transaction) -> float:
    """Calculate fraud risk score."""
    # Automatically creates span with args/result
    score = analyze_transaction(transaction)
    return score

# ✅ Don't trace functions with sensitive data in results
@traced(capture_result=False)
async def decrypt_credentials(encrypted: str) -> Credentials:
    """Decrypt user credentials."""
    # Span created but result not captured
    return decrypt(encrypted)

# ✅ Manual span creation for complex operations
async def process_batch(items: List[Item]) -> BatchResult:
    """Process multiple items with detailed tracing."""
    with create_span("validate-batch", {"item_count": len(items)}) as span:
        validate_items(items)
        span.set_attribute("validation.passed", True)

    with create_span("process-items") as span:
        results = []
        for item in items:
            result = await process_item(item)
            results.append(result)
        span.set_attribute("processed_count", len(results))

    return BatchResult(results=results)
```

## Error Handling Standards

### Activity Error Handling

```python
@activity.defn(name="call_external_api")
async def call_external_api(request: ApiRequest) -> ApiResponse:
    """Call external API with proper error handling."""
    try:
        response = await http_client.post(url=request.url, timeout=30.0)
        response.raise_for_status()
        return ApiResponse.from_json(response.json())
    except httpx.TimeoutException:
        activity.logger.error("API timeout")
        raise  # Let Temporal retry
    except httpx.HTTPStatusError as e:
        if e.response.status_code >= 500:
            raise  # Retryable
        else:
            raise ApplicationError(str(e), non_retryable=True)
    except Exception as e:
        activity.logger.exception(f"Unexpected error: {e}")
        raise ApplicationError(str(e), non_retryable=True)
```

### Workflow Error Handling

```python
@workflow.run
async def run(self, input: CombinedInput) -> str:
    """Main workflow with error handling."""
    while True:
        await workflow.wait_condition(lambda: bool(self.prompt_queue) or self.chat_ended)
        if self.chat_should_end():
            return str(self.conversation_history)

        try:
            result = await workflow.execute_activity_method(
                ToolActivities.agent_tool_planner,
                args=[self.prompt_queue.popleft()],
                start_to_close_timeout=LLM_ACTIVITY_START_TO_CLOSE_TIMEOUT,
                retry_policy=RetryPolicy(initial_interval=timedelta(seconds=5))
            )
        except ActivityError as e:
            workflow.logger.error(f"Activity failed: {e}")
            self.add_message("agent", {"response": "Error. Please retry."})
            continue
```

## Agent & Tool Standards

### Goal Definition

```python
goal_travel = AgentGoal(
    id="goal_travel_booking",
    category_tag="travel",
    agent_name="Travel Assistant",
    agent_friendly_description="Book flights and hotels",
    tools=[search_flights_tool, book_flights_tool],
    description="Help users book travel: 1. SearchFlights 2. BookFlights",
    starter_prompt="Hi! I can help you book flights and hotels. Where to?",
    example_conversation_history="user: Fly to Paris\nagent: Searching...",
    mcp_server_definition=None
)
```

### Tool Definition

```python
search_flights_tool = ToolDefinition(
    name="SearchFlights",
    description="Search flights between two cities. Ask user to confirm dates first.",
    arguments=[
        ToolArgument(name="origin", type="string", description="Airport code (JFK, LAX)", required=True),
        ToolArgument(name="destination", type="string", description="Airport code (CDG, LHR)", required=True),
        ToolArgument(name="departure_date", type="ISO8601", description="YYYY-MM-DD", required=True),
    ]
)
```

### Tool Implementation

```python
@activity.defn(name="search_flights")
async def search_flights(origin: str, destination: str, departure_date: str) -> List[Flight]:
    """Search for flights between two airports."""
    if not origin or not destination:
        raise ApplicationError("Invalid airports", non_retryable=True)

    try:
        results = await flight_api.search(
            from_airport=origin.upper(),
            to_airport=destination.upper(),
            depart=departure_date
        )
        return [Flight(**f) for f in results["flights"]]
    except Exception as e:
        activity.logger.error(f"Flight search failed: {e}")
        raise ApplicationError(f"Unable to search flights: {e}", non_retryable=True)
```

## Testing Standards

### Unit Test Structure

```python
@pytest.mark.asyncio
async def test_agent_tool_planner_success():
    """Test successful LLM planning call."""
    input_data = ToolPromptInput(prompt="Book a flight", context="...")
    expected = {"next": "confirm", "tool": "SearchFlights", "args": {...}}

    with patch('activities.tool_activities.llm_client') as mock_llm:
        mock_llm.call.return_value = expected
        result = await ToolActivities.agent_tool_planner(input_data)
        assert result == expected
        mock_llm.call.assert_called_once()
```

### Temporal Workflow Tests

```python
@pytest.mark.asyncio
async def test_agent_workflow_conversation():
    """Test basic workflow conversation flow."""
    async with await WorkflowEnvironment.start_time_skipping() as env:
        async with Worker(env.client, task_queue="test", workflows=[AgentGoalWorkflow]):
            handle = await env.client.start_workflow(
                AgentGoalWorkflow.run,
                CombinedInput(tool_params=AgentGoalWorkflowParams(), agent_goal=test_goal),
                id="test-workflow",
                task_queue="test"
            )
            await handle.signal("user_prompt", "Hello")
            history = await handle.query("get_conversation_history")
            assert len(history["messages"]) > 0
```

## Documentation Standards

### Code Documentation

**Function docstrings:**
```python
def calculate_price(
    base_price: float,
    quantity: int,
    discount: Optional[float] = None
) -> float:
    """Calculate final price with optional discount.

    Args:
        base_price: Base price per unit
        quantity: Number of units
        discount: Optional discount percentage (0-100)

    Returns:
        Final price after discount

    Raises:
        ValueError: If base_price or quantity is negative
        ValueError: If discount is not in range 0-100

    Example:
        >>> calculate_price(10.0, 5)
        50.0
        >>> calculate_price(10.0, 5, discount=20.0)
        40.0
    """
    if base_price < 0 or quantity < 0:
        raise ValueError("Price and quantity must be non-negative")

    if discount is not None and not (0 <= discount <= 100):
        raise ValueError("Discount must be between 0 and 100")

    total = base_price * quantity
    if discount:
        total *= (1 - discount / 100)
    return total
```

**Class docstrings:**
```python
class AgentGoalWorkflow:
    """Temporal workflow for conversational AI agents.

    Manages conversation state, validates user input, calls LLM for planning,
    and executes tools after user confirmation. Supports goal switching and
    continue-as-new for long conversations.

    Signals:
        user_prompt: Receive user message
        confirm: Confirm tool execution
        end_chat: End conversation

    Queries:
        get_conversation_history: Get full conversation history
        get_agent_goal: Get current agent goal
        get_latest_tool_data: Get latest tool call details

    Example:
        # Start workflow
        handle = await client.start_workflow(
            AgentGoalWorkflow.run,
            CombinedInput(...),
            id="agent-workflow",
            task_queue="agent-queue"
        )

        # Send user message
        await handle.signal("user_prompt", "Book a flight")

        # Get conversation history
        history = await handle.query("get_conversation_history")
    """
```

### README Structure

Each module should have clear documentation:
```markdown
# Module Name

Brief description (1-2 sentences).

## Purpose

Detailed explanation of what this module does.

## Usage

```python
# Example code
from module import function
result = function(arg1, arg2)
```

## API Reference

### `function_name(arg1: type, arg2: type) -> return_type`

Description of what the function does.

**Parameters:**
- `arg1` - Description
- `arg2` - Description

**Returns:**
- Description of return value

**Raises:**
- `ErrorType` - When this error occurs
```

## Security Standards

### Secrets Management

- Always use environment variables (`os.getenv()`)
- Never hardcode secrets in code
- Use `.env` file (not in repo) for local development
- Use secrets manager for production (AWS Secrets Manager, Vault)

### Input Validation

```python
def transfer_money(from_account: str, to_account: str, amount: float):
    if not from_account or amount <= 0:
        raise ValueError("Invalid input")
    if amount > 10000:
        raise ValueError("Amount exceeds limit")
```

### Logging Sensitive Data

- Redact PII in logs: `user_id[:4]****`
- Never log credit cards, API keys, passwords
- Use OpenBox guardrails for automatic redaction via `@traced(capture_result=False)`

## Performance Standards

- **Batch operations:** Fetch multiple items in single API/DB call
- **Parallel activities:** Use `asyncio.gather()` for independent operations
- **Connection pooling:** Reuse HTTP/DB connections
- **Avoid N+1:** Load all needed data in one query

## Git Standards

### Commit Messages

Use conventional commits: `<type>(<scope>): <subject>`

**Types:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

**Example:**
```
feat(workflows): add continue-as-new support

Implement continue-as-new pattern for long conversations.
Resets after 250 turns while preserving summary.
```

### Branch Naming

- Feature: `feature/short-description`
- Bug fix: `fix/short-description`
- Hotfix: `hotfix/short-description`
- Docs: `docs/short-description`

**Examples:**
- `feature/mcp-integration`
- `fix/workflow-timeout`
- `docs/api-reference`

## References

- **PEP 8:** https://peps.python.org/pep-0008/
- **Black:** https://black.readthedocs.io/
- **Temporal Best Practices:** https://docs.temporal.io/dev-guide/python/best-practices
- **OpenBox SDK Docs:** See project README.md
