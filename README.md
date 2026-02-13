# Temporal AI Agent - OpenBox SDK Demo

A demo project showing how to integrate the [OpenBox SDK](https://pypi.org/project/openbox-temporal-sdk-python/) into a [Temporal AI Agent]([https://github.com/temporalio/temporal-ai-agent](https://github.com/temporal-community/temporal-ai-agent)) application. The agent uses an LLM to chat with users, gather requirements, and execute tools (flight booking, invoicing, HR tasks, etc.) — all orchestrated durably by Temporal, with OpenBox providing governance and observability.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      React Frontend                         │
│                    (localhost:5173)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                      FastAPI Server                         │
│                    (localhost:8000)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Temporal Worker                           │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  Workflow    │  │  Activities  │  │  OpenBox SDK       │  │
│  │  (LLM loop, │  │  (LLM calls, │  │  (governance,      │  │
│  │   signals,   │  │   tool exec) │  │   observability,   │  │
│  │   goals)     │  │              │  │   guardrails)      │  │
│  └─────────────┘  └──────────────┘  └─────────┬─────────┘  │
└───────────────────────────────────────────────┬─────────────┘
                                                │
                                   ┌────────────▼────────────┐
                                   │     OpenBox Core        │
                                   │  (policy evaluation,    │
                                   │   governance actions)   │
                                   └─────────────────────────┘
```

## Prerequisites

- Python 3.10+
- [uv](https://docs.astral.sh/uv/) (Python package manager)
- Node.js 18+ and npm
- A running [Temporal server](https://learn.temporal.io/getting_started/python/dev_environment/) (local or cloud)
- An LLM API key (OpenAI, Anthropic, Google, etc.)

## Quick Start

### 1. Install dependencies

```bash
make setup
```

Or manually:
```bash
uv sync
cd frontend && npm install
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and set at minimum:
```bash
LLM_MODEL=openai/gpt-4o       # or anthropic/claude-3-sonnet, gemini/gemini-2.0-flash, etc.
LLM_KEY=your-api-key

TEMPORAL_ADDRESS=localhost:7233 # default
```

### 3. Start Temporal (if not running)

```bash
brew install temporal
temporal server start-dev
```

### 4. Run the application

In separate terminals:

```bash
# Terminal 1: Temporal worker
make run-worker

# Terminal 2: API server
make run-api

# Terminal 3: Frontend
make run-frontend
```

Or all at once:
```bash
make run-dev
```

### 5. Open the UI

Visit [http://localhost:5173](http://localhost:5173)

## OpenBox SDK Integration

This demo uses the `openbox-temporal-sdk-python` package to add governance and observability to the Temporal worker. The integration point is in `scripts/run_worker.py`:

```python
from openbox import create_openbox_worker

worker = create_openbox_worker(
    client=client,
    task_queue=TEMPORAL_TASK_QUEUE,
    workflows=[AgentGoalWorkflow],
    activities=[...],
    # OpenBox config
    openbox_url=os.getenv("OPENBOX_URL"),
    openbox_api_key=os.getenv("OPENBOX_API_KEY"),
    governance_timeout=float(os.getenv("OPENBOX_GOVERNANCE_TIMEOUT", "30.0")),
    governance_policy=os.getenv("OPENBOX_GOVERNANCE_POLICY", "fail_open"),
    instrument_databases=True,
    instrument_file_io=True,
)
```

Configure OpenBox in your `.env`:

```bash
OPENBOX_URL=https://openbox-core-v2.node.lat
OPENBOX_API_KEY=your-openbox-api-key
OPENBOX_GOVERNANCE_ENABLED=true
OPENBOX_GOVERNANCE_POLICY=fail_closed  # fail_open or fail_closed
```

For full SDK documentation, see the [OpenBox SDK docs](https://pypi.org/project/openbox-temporal-sdk-python/).

## Available Agent Goals

The agent supports multiple goals, configured via `AGENT_GOAL` in `.env`:

| Goal ID | Category | Description |
|---------|----------|-------------|
| `goal_event_flight_invoice` | Travel | Find events in AU/NZ, book flights, generate invoice (default) |
| `goal_match_train_invoice` | Travel | Find Premier League matches, book trains, generate invoice |
| `goal_fin_check_account_balances` | Finance | Check account balances |
| `goal_fin_move_money` | Finance | Initiate money transfers |
| `goal_fin_loan_application` | Finance | Submit a loan application |
| `goal_hr_schedule_pto` | HR | Schedule paid time off |
| `goal_hr_check_pto` | HR | Check PTO balance |
| `goal_ecomm_order_status` | E-commerce | Check order status and tracking |
| `goal_ecomm_list_orders` | E-commerce | List all orders |
| `goal_food_ordering` | Food | Order food with Stripe payment (MCP) |
| `goal_choose_agent_type` | System | Multi-agent mode: let user pick a goal |

### Multi-Agent Mode

Set `AGENT_GOAL=goal_choose_agent_type` to let users choose their agent at runtime. Control which categories appear with `GOAL_CATEGORIES`:

```bash
GOAL_CATEGORIES=all  # or: hr,travel-flights,travel-trains,fin,ecommerce,food
```

## Goal-Specific Configuration

### Travel - Flights (`goal_event_flight_invoice`)

- **Events**: Mock data, no configuration needed
- **Flights**: Generates realistic mock data by default. Set `RAPIDAPI_KEY` for live data from [RapidAPI Sky Scrapper](https://rapidapi.com/apiheya/api/sky-scrapper)
- **Invoice**: Set `STRIPE_API_KEY` for real Stripe invoices, or leave unset for mock invoices

### Travel - Trains (`goal_match_train_invoice`)

- **Fixtures**: Mock data by default. Set `FOOTBALL_DATA_API_KEY` for real Premier League data from [Football Data](https://www.football-data.org)
- **Trains**: Start the train API: `make run-train-api`
- **Invoice**: Requires `STRIPE_API_KEY` (no mock fallback for this goal)
- **Note**: This goal has intentional failure built in to demonstrate Temporal retry behavior. See [docs/setup.md](docs/setup.md) for details on the .NET enterprise worker

### Finance Goals

Edit mock user data in `tools/data/customer_account_data.json`. Set `FIN_START_REAL_WORKFLOW=TRUE` to initiate real Temporal workflows for money movement/loans.

### HR Goals

Edit mock PTO data in `tools/data/employee_pto_data.json`.

### E-commerce Goals

Edit mock order data in `tools/data/customer_order_data.json`.

### Food Ordering (`goal_food_ordering`)

Requires `STRIPE_API_KEY` and Stripe products with metadata `use_case=food_ordering_demo`. Run `tools/food/setup/create_stripe_products.py` to seed menu items.

## LLM Configuration

This project uses [LiteLLM](https://docs.litellm.ai/docs/providers) to support multiple LLM providers:

```bash
# OpenAI
LLM_MODEL=openai/gpt-4o
LLM_KEY=sk-...

# Anthropic
LLM_MODEL=anthropic/claude-3-sonnet
LLM_KEY=sk-ant-...

# Google
LLM_MODEL=gemini/gemini-2.0-flash
LLM_KEY=...

# Ollama (local)
LLM_MODEL=ollama/mistral
LLM_BASE_URL=http://localhost:11434
```

## Docker

```bash
# Development (with hot-reload)
docker compose up -d

# Production
docker compose -f docker-compose.yml up -d
```

Default URLs:
- Temporal UI: [http://localhost:8080](http://localhost:8080)
- API: [http://localhost:8000](http://localhost:8000)
- Frontend: [http://localhost:5173](http://localhost:5173)

## Project Structure

```
├── workflows/          # Temporal workflow definitions
├── activities/         # Temporal activity functions
├── tools/              # Agent tool implementations
├── goals/              # Goal configurations and prompts
├── prompts/            # LLM prompt generators
├── models/             # Data models and types
├── api/                # FastAPI backend
├── frontend/           # React/Vite frontend
├── shared/             # Shared utilities and config
├── scripts/            # Worker and utility scripts
├── tests/              # Test suite
├── docs/               # Additional documentation
├── thirdparty/         # Third-party service mocks (train API)
└── enterprise/         # .NET enterprise worker (C#)
```

## Further Documentation

- [Architecture](docs/architecture.md) - System components and interactions
- [Setup Guide](docs/setup.md) - Detailed configuration and running instructions
- [Adding Goals and Tools](docs/adding-goals-and-tools.md) - Extending the agent
- [Architecture Decisions](docs/architecture-decisions.md) - Design rationale
- [Contributing](docs/contributing.md) - How to contribute and run tests

## License

MIT
