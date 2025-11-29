# Repository Architecture Documentation

## Overview

This repository contains a production-ready **Slack bot** that helps users decide what to order for dinner. The application demonstrates a sophisticated integration of modern Python technologies:

- **PydanticAI** - Type-safe AI agents with structured outputs
- **Temporal** - Durable workflow orchestration for long-running conversations
- **FastAPI** - High-performance web framework for handling Slack webhooks
- **Slack SDK** - Integration with Slack Events API and Web API

The system uses a **multi-agent architecture** where a fast dispatcher agent routes requests and a specialized research agent performs web searches to find restaurant recommendations. **Temporal workflows** manage entire conversation threads, maintaining state across multiple user interactions and surviving server restarts. All external API calls (Slack, OpenAI, DuckDuckGo) are isolated as **Temporal activities** for fault tolerance and automatic retries.

**Key Architectural Highlights:**
- Event-driven design with Slack webhook integration
- Durable execution with workflow-based conversation management
- Type-safe data models throughout (Pydantic)
- Comprehensive observability via Logfire instrumentation
- HMAC signature verification for webhook security
- Incremental message fetching with cursor-based pagination

---

## Quick Start Guide

### How to Use This Documentation

This documentation set is organized to serve both newcomers and experienced developers:

1. **Start here (README.md)** - Get a high-level understanding of the entire system
2. **Component Inventory** - Understand what each Python module does and where to find specific code
3. **Architecture Diagrams** - Visualize system structure and component relationships
4. **Data Flows** - Trace how data moves through the system in various scenarios
5. **API Reference** - Look up specific functions, classes, and configuration options

**Navigation Tips:**
- All file references include absolute paths and line numbers for easy lookup
- Diagrams use Mermaid format (viewable in GitHub, VS Code, and most markdown viewers)
- Code examples are provided for all major patterns
- Cross-references link related documentation sections

### Documentation Map

| Document | Purpose | Best For |
|----------|---------|----------|
| [README.md](./README.md) | High-level architecture overview and synthesis | First-time exploration, understanding big picture |
| [01_component_inventory.md](./docs/01_component_inventory.md) | Complete catalog of all modules, classes, functions | Finding specific code, understanding module purposes |
| [02_architecture_diagrams.md](./diagrams/02_architecture_diagrams.md) | Visual diagrams of system architecture | Understanding component relationships, system design |
| [03_data_flows.md](./docs/03_data_flows.md) | Sequence diagrams and flow analysis | Debugging, understanding request lifecycles, tracing data |
| [04_api_reference.md](./docs/04_api_reference.md) | Detailed API documentation with examples | Development, integration, configuration |

---

## Architecture Summary

### System Architecture

The application follows a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    External Systems                          │
│  Slack Platform │ OpenAI API │ DuckDuckGo │ Logfire         │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (FastAPI)                     │
│  • /slack-events endpoint                                    │
│  • HMAC signature verification                               │
│  • Event type discrimination                                 │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│            Temporal Orchestration Layer                      │
│  • SlackThreadWorkflow (one per conversation thread)         │
│  • Workflow signals for event routing                        │
│  • Durable state management                                  │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
           ┌──────────────────┴──────────────────┐
           │                                     │
┌──────────────────────┐              ┌─────────────────────┐
│   Activity Layer     │              │   Agent Layer       │
│  • Slack API calls   │              │  • Dispatch agent   │
│  • Message posting   │              │  • Research agent   │
│  • Reactions         │              │  • Tool execution   │
│  • Thread fetching   │              │  • LLM calls        │
└──────────────────────┘              └─────────────────────┘
```

**Data Flow Overview:**
1. User mentions bot in Slack → Webhook delivered to FastAPI
2. FastAPI validates signature → Creates/signals Temporal workflow
3. Workflow fetches thread messages → Runs dispatcher agent
4. Dispatcher analyzes context → Either responds directly or delegates to research
5. Research agent searches web → Formats results
6. Workflow posts response to Slack → Continues waiting for next message

### Key Design Decisions

#### 1. Multi-Agent Architecture

**Decision:** Use two specialized agents instead of one general-purpose agent.

**Rationale:**
- **Dispatcher Agent (gpt-5-mini)**: Fast, inexpensive, handles routing and information gathering
- **Research Agent (gpt-5-mini)**: Focused on web search and restaurant recommendations
- Allows for different prompts, tools, and output structures per agent
- Reduces cost by using lightweight model for simple tasks

**Implementation:** `pydantic_temporal_example/agents/`

#### 2. Temporal for Workflow Orchestration

**Decision:** Use Temporal instead of stateless webhook handlers.

**Rationale:**
- Maintains conversation state across multiple messages
- Survives server crashes and restarts
- Provides built-in retry logic for activities
- Enables complex multi-turn conversations without external state management
- **One workflow per Slack thread** for isolated conversation contexts

**Implementation:** `pydantic_temporal_example/temporal/workflows.py`

#### 3. Structured Agent Outputs

**Decision:** Use Pydantic dataclasses with discriminated unions for agent outputs.

**Rationale:**
- Type-safe responses at compile time and runtime
- Discriminated unions enable pattern matching on agent decisions
- Self-documenting through docstrings on fields
- Prevents parsing errors from free-text responses

**Example:**
```python
# Agent can return one of three types (discriminated by "type" field)
DispatchResult = NoResponse | SlackResponse | DinnerOptionResearchRequest
```

**Implementation:** `pydantic_temporal_example/agents/dispatch_agent.py` (lines 9-62)

#### 4. Activity-Based External I/O

**Decision:** Isolate all Slack API calls as Temporal activities.

**Rationale:**
- Activities are automatically retriable on failure
- Timeouts prevent hanging workflows
- Activities can be tested independently
- Clear separation between orchestration (workflow) and execution (activity)

**Implementation:** `pydantic_temporal_example/temporal/slack_activities.py`

#### 5. Incremental Message Fetching

**Decision:** Track `_most_recent_ts` and only fetch new messages.

**Rationale:**
- Reduces API calls to Slack
- Minimizes data transfer
- Improves performance for long threads
- Prevents re-processing old messages

**Implementation:** `pydantic_temporal_example/temporal/workflows.py` (lines 41-46, 76-80)

### Technology Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Web Framework** | FastAPI | 0.115.0+ | HTTP server, webhook handling, dependency injection |
| **AI Framework** | PydanticAI | 1.0.2+ | Agent orchestration, structured outputs, tool integration |
| **Workflow Engine** | Temporal | Latest | Durable execution, state management, fault tolerance |
| **Data Validation** | Pydantic | Latest | Runtime type validation, settings management |
| **Chat Platform** | Slack SDK | 3.36.0+ | Slack API integration, webhook verification |
| **LLM Provider** | OpenAI | Latest | GPT-5-mini for agent inference |
| **Search** | DuckDuckGo | Latest | Web search via PydanticAI tools |
| **Observability** | Logfire | 4.3.4+ | Tracing, logging, performance monitoring |
| **Language** | Python | 3.12+ | Application development |
| **Server** | Uvicorn | 0.30.0+ | ASGI server for FastAPI |

---

## Component Overview

### Core Components

The system consists of **15 Python modules** organized into a structured package:

#### Package Structure
```
pydantic_temporal_example/
├── __init__.py                    # Package marker
├── app.py                         # Application entry point
├── api.py                         # FastAPI routes and event handlers
├── models.py                      # Pydantic data models
├── settings.py                    # Configuration management
├── dependencies.py                # FastAPI dependency injection
├── slack.py                       # Webhook signature verification
├── agents/
│   ├── __init__.py
│   ├── dispatch_agent.py         # Intent classification agent
│   └── dinner_research_agent.py  # Restaurant research agent
└── temporal/
    ├── __init__.py
    ├── workflows.py              # SlackThreadWorkflow definition
    ├── activities.py             # Slack API activity implementations
    ├── worker.py                 # Temporal worker setup
    └── client.py                 # Temporal client builder
```

#### Module Breakdown

**Entry Point:**
- `app.py` - Main application with FastAPI setup, Logfire configuration, and execution entry point

**API Layer:**
- `api.py` - Single POST endpoint `/slack-events` with event discrimination logic
- `slack.py` - HMAC-SHA256 signature verification and timestamp validation
- `dependencies.py` - Lifespan management, Temporal client injection, bot user ID injection

**Data Layer:**
- `models.py` - 9 Pydantic models for Slack events, messages, and agent I/O
- `settings.py` - Environment-based configuration with caching

**Temporal Layer:**
- `workflows.py` - SlackThreadWorkflow class with signal handlers
- `slack_activities.py` - 5 activities for Slack API operations
- `worker.py` - Worker initialization with plugin registration
- `client.py` - Client builder with Logfire and PydanticAI plugins

**Agent Layer:**
- `dispatch_agent.py` - Fast router agent with 3 output types
- `dinner_research_agent.py` - Research agent with DuckDuckGo tool

**Statistics:**
- **Total Files:** 15 Python modules
- **Total Classes:** 16 (11 Pydantic models, 1 workflow, 1 settings, 3 event types)
- **Total Functions:** 15 (2 public API endpoints, 13 internal)
- **Temporal Components:** 1 workflow, 5 activities, 2 agents
- **API Endpoints:** 1 POST route

### Component Interactions

```mermaid
flowchart LR
    Slack[Slack Events] --> API[api.py]
    API --> Verify[slack.py]
    API --> Temporal[workflows.py]
    Temporal --> Activities[slack_activities.py]
    Temporal --> Dispatch[dispatch_agent.py]
    Temporal --> Dinner[dinner_research_agent.py]
    Activities --> Slack
    Dispatch --> OpenAI[OpenAI API]
    Dinner --> OpenAI
    Dinner --> DDG[DuckDuckGo]

    Models[models.py] -.-> API
    Models -.-> Temporal
    Models -.-> Activities
    Settings[settings.py] -.-> API
    Settings -.-> Activities
    Settings -.-> Worker[worker.py]
```

**Key Interaction Patterns:**

1. **Slack → FastAPI → Temporal**
   - Slack webhook arrives at `/slack-events`
   - Request validated and parsed
   - Workflow created (app_mention) or signaled (message)

2. **Temporal → Activities → Slack**
   - Workflow schedules activities
   - Activities make Slack API calls
   - Results returned to workflow

3. **Temporal → Agents → External Services**
   - Workflow runs agents via TemporalAgent wrappers
   - Agents call OpenAI for inference
   - Agents use DuckDuckGo tool for searches
   - Results returned to workflow

4. **Configuration Flow**
   - Settings loaded from environment
   - Injected via FastAPI dependencies
   - Used across all layers

### File Structure

```

├── pydantic_temporal_example/          # Main package
│   ├── agents/                         # AI agent definitions
│   │   ├── dispatch_agent.py          # Lines 64-94: Agent config
│   │   └── dinner_research_agent.py   # Lines 35-43: Agent config
│   ├── temporal/                      # Temporal components
│   │   ├── workflows.py               # Lines 35-108: SlackThreadWorkflow
│   │   ├── slack_activities.py        # Lines 12-81: 5 activities
│   │   ├── worker.py                  # Lines 17-36: Worker context manager
│   │   └── client.py                  # Lines 8-21: Client builder
│   ├── app.py                         # Lines 9-22: App setup + main()
│   ├── api.py                         # Lines 23-78: Event handlers
│   ├── models.py                      # Lines 7-92: 9 Pydantic models
│   ├── settings.py                    # Lines 6-17: Config
│   ├── dependencies.py                # Lines 14-28: DI functions
│   └── slack.py                       # Lines 13-48: Signature verification
├── pyproject.toml                     # Dependencies and project config
├── uv.lock                           # Lock file
└── README.md                         # Project documentation
```

---

## Data Flow Patterns

### Request Lifecycle

**Complete flow from Slack mention to bot response:**

1. **Event Reception (API Layer)**
   - Slack sends POST to `/slack-events`
   - Headers: `x-slack-signature`, `x-slack-request-timestamp`
   - Body: JSON with event payload

2. **Validation (Security Layer)**
   - Extract timestamp, verify not older than 5 minutes
   - Extract signature, compute expected HMAC-SHA256
   - Compare signatures using constant-time comparison
   - Parse JSON to Pydantic models with discriminator

3. **Routing (API Layer)**
   - If `URLVerificationEvent` → Echo challenge
   - If `AppMentionEvent` → Create workflow
   - If `MessageChannelsEvent` + bot's message → Ignore
   - If `MessageChannelsEvent` + user message → Signal workflow

4. **Workflow Creation/Signaling (Orchestration Layer)**
   - Generate workflow ID from thread timestamp
   - Start workflow with initial signal OR
   - Get workflow handle and send signal

5. **Event Processing (Workflow)**
   - Queue event in `_pending_events`
   - Add "spin" reaction emoji (thinking indicator)
   - Fetch new messages from thread
   - Update `_thread_messages` history
   - Stringify thread for agent context

6. **Agent Orchestration (Agent Layer)**
   - Run dispatcher agent with thread context
   - Agent analyzes and returns one of:
     - `NoResponse` → Exit without posting
     - `SlackResponse` → Post directly
     - `DinnerOptionResearchRequest` → Run research agent

7. **Research (If Delegated)**
   - Extract user preferences from dispatch request
   - Run research agent with preferences
   - Agent searches web via DuckDuckGo
   - Agent formats results with OpenAI
   - Returns `DinnerSuggestions`

8. **Response Posting (Activity Layer)**
   - Parallel execution:
     - Remove "spin" reaction
     - Post response message to thread
   - Return 204 to Slack

### Key Flows

#### Flow 1: Simple Query (User has all info)

```
User: "@bot I want Thai food under $25 in Seattle"
  ↓
Webhook → Verify → Create Workflow → Fetch Thread
  ↓
Dispatcher: Sees complete info → Returns DinnerOptionResearchRequest
  ↓
Research Agent: Searches "Thai restaurants Seattle under $25"
  ↓
OpenAI: Formats results as DinnerSuggestions
  ↓
Post to Slack: "Here are 3 Thai restaurants..."
```

**Files Involved:**
- `api.py` (lines 55-65) - Workflow creation
- `workflows.py` (lines 64-107) - Event handling
- `workflows.py` (lines 110-118) - Agent orchestration
- `dispatch_agent.py` (lines 64-94) - Intent classification
- `dinner_research_agent.py` (lines 35-43) - Research execution

#### Flow 2: Multi-Turn Conversation (Missing info)

```
User: "@bot help with dinner"
  ↓
Workflow Created → Dispatcher: Missing location
  ↓
Bot: "What city are you in?"
  ↓
User: "Boston"
  ↓
Signal Workflow → Dispatcher: Missing cuisine
  ↓
Bot: "What kind of food do you like?"
  ↓
User: "Italian, vegetarian, mid-range"
  ↓
Signal Workflow → Dispatcher: Has all info → Delegates
  ↓
Research Agent → Search → Format
  ↓
Bot: "Here are 3 Italian restaurants in Boston..."
```

**Files Involved:**
- `workflows.py` (lines 60-62) - Message signal handler
- `workflows.py` (lines 37-46) - State management
- `slack_activities.py` (lines 12-28) - Incremental message fetching

#### Flow 3: Error Handling (Rate Limit)

```
Workflow → Execute activity: slack_chat_post_message
  ↓
Slack API: 429 Too Many Requests
  ↓
Activity Fails → Temporal: Apply retry policy
  ↓
Wait (exponential backoff)
  ↓
Retry Activity → Slack API: Success
  ↓
Workflow Continues
```

**Files Involved:**
- `slack_activities.py` (lines 31-40) - Activity definition
- `workflows.py` (lines 102-106) - Activity execution with timeout

---

## API Quick Reference

### Endpoints

| Method | Path | Purpose | Authentication |
|--------|------|---------|----------------|
| POST | `/slack-events` | Receive Slack webhook events | HMAC signature |

**Request Headers:**
- `x-slack-request-timestamp` (required) - Request timestamp for replay protection
- `x-slack-signature` (required) - HMAC-SHA256 signature

**Response Codes:**
- `200 OK` - URL verification challenge response
- `204 No Content` - Event processed successfully
- `401 Unauthorized` - Invalid signature or timestamp
- `422 Unprocessable Entity` - Invalid event payload

### Key Classes

#### Event Models

```python
# Slack sends an app mention
AppMentionEvent(
    type="app_mention",
    user="U12345",
    text="@bot help",
    ts="1234567890.123456",
    channel="C12345",
    event_ts="1234567890.123456",
    thread_ts=None  # or thread timestamp
)

# User replies in thread
MessageChannelsEvent(
    type="message",
    user="U12345",
    text="Seattle",
    ts="1234567891.123456",
    channel="C12345",
    event_ts="1234567891.123456",
    channel_type="channel",
    thread_ts="1234567890.123456"
)
```

#### Agent Output Models

```python
# Dispatcher decides no response needed
NoResponse(type="no-response")

# Dispatcher asks clarifying question
SlackResponse(
    type="slack-response",
    response="What city are you in?"
)

# Dispatcher delegates to research
DinnerOptionResearchRequest(
    type="dinner-option-research-request",
    location=WebSearchUserLocation(city="Seattle", state="WA"),
    cuisine_preferences="Thai",
    price_preferences="under $25",
    dietary_restrictions="none",
    extra_info=None
)

# Research agent returns results
DinnerSuggestions(
    response="Here are 3 Thai restaurants..."  # or Block Kit blocks
)
```

### Configuration

**Environment Variables:**

```bash
# Required
SLACK_BOT_TOKEN=xoxb-your-token-here
SLACK_SIGNING_SECRET=your-signing-secret
OPENAI_API_KEY=sk-your-openai-key

# Optional
TEMPORAL_HOST=  # Empty = start local dev server
TEMPORAL_PORT=7233
TEMPORAL_TASK_QUEUE=agent-task-queue
```

**Usage:**

```python
from pydantic_temporal_example.settings import get_settings

settings = get_settings()  # Cached singleton
print(settings.slack_bot_token)
```

---

## Getting Started

### Prerequisites

**Required:**
- Python 3.12+
- Slack workspace with admin access
- OpenAI API key
- Temporal server (or use built-in dev server)

**System Dependencies:**
- `uv` package manager (recommended) or `pip`
- Modern OS (Linux, macOS, WSL2 on Windows)

### Environment Setup

1. **Clone Repository:**
   ```bash
   git clone <repository-url>
   cd pydantic-ai-temporal-example
   ```

2. **Install Dependencies:**
   ```bash
   uv sync
   # or with pip:
   # pip install -e .
   ```

3. **Configure Slack App:**
   - Create Slack app at https://api.slack.com/apps
   - Enable Event Subscriptions
   - Subscribe to `app_mention` and `message.channels` events
   - Install app to workspace
   - Copy Bot OAuth Token and Signing Secret

4. **Set Environment Variables:**
   ```bash
   # Create .env file
   cat > .env << EOF
   SLACK_BOT_TOKEN=xoxb-your-token
   SLACK_SIGNING_SECRET=your-secret
   OPENAI_API_KEY=sk-your-key
   TEMPORAL_HOST=
   EOF
   ```

5. **Configure Webhook URL:**
   - Run application (see below)
   - Expose port 4000 via ngrok or similar:
     ```bash
     ngrok http 4000
     ```
   - Set Request URL in Slack app: `https://your-domain/slack-events`

### Running the Application

**Using uv (recommended):**
```bash
uv run python -m pydantic_temporal_example.app
```

**Using Python directly:**
```bash
python -m pydantic_temporal_example.app
```

**What Starts:**
1. Temporal worker (with local dev server if `TEMPORAL_HOST` is empty)
2. FastAPI server on port 4000
3. Logfire instrumentation
4. Workflow and activity registration
5. Agent plugin initialization

**Expected Output:**
```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:4000
```

**Testing:**
1. Mention bot in Slack: `@yourbot help with dinner`
2. Bot should respond asking for more information
3. Reply with location and preferences
4. Bot provides restaurant suggestions

---

## Development Guide

### Adding New Features

#### Adding a New Agent

1. **Create Agent Module:**
   ```python
   # pydantic_temporal_example/agents/new_agent.py
   from pydantic_ai import Agent
   from dataclasses import dataclass

   @dataclass
   class NewAgentOutput:
       result: str

   new_agent = Agent(
       model="openai-responses:gpt-5-mini",
       output_type=NewAgentOutput,
       instructions="Agent instructions...",
       tools=[]
   )
   ```

2. **Create Temporal Wrapper:**
   ```python
   # In workflows.py
   from pydantic_ai.durable_exec.temporal import TemporalAgent
   from .agents.new_agent import new_agent

   temporal_new_agent = TemporalAgent(new_agent, name="new_agent")
   ```

3. **Register in Worker:**
   ```python
   # In worker.py, add to plugins list
   plugins=[
       AgentPlugin(temporal_dispatch_agent),
       AgentPlugin(temporal_dinner_research_agent),
       AgentPlugin(temporal_new_agent)  # Add here
   ]
   ```

4. **Use in Workflow:**
   ```python
   result = await temporal_new_agent.run(input_data)
   ```

#### Adding a New Activity

1. **Define Activity:**
   ```python
   # In slack_activities.py
   @activity.defn
   @logfire.instrument
   async def new_slack_activity(param: NewParam) -> NewResult:
       client = _get_slack_client()
       response = await client.some_api_method(...)
       return response.data
   ```

2. **Add to Registry:**
   ```python
   # In slack_activities.py
   ALL_SLACK_ACTIVITIES = [
       slack_conversations_replies,
       slack_chat_post_message,
       slack_chat_delete,
       slack_reactions_add,
       slack_reactions_remove,
       new_slack_activity  # Add here
   ]
   ```

3. **Use in Workflow:**
   ```python
   result = await workflow.execute_activity(
       new_slack_activity,
       param,
       start_to_close_timeout=timedelta(seconds=10)
   )
   ```

#### Adding a New Event Type

1. **Define Pydantic Model:**
   ```python
   # In models.py
   class NewEvent(BaseModel):
       type: Literal["new_event_type"]
       # ... fields
   ```

2. **Update Type Union:**
   ```python
   SlackEvent: TypeAlias = Annotated[
       AppMentionEvent | MessageChannelsEvent | NewEvent,
       Discriminator("type")
   ]
   ```

3. **Add Handler:**
   ```python
   # In api.py
   async def handle_new_event(event: NewEvent, client: TemporalClient):
       # Handle event
       pass

   # In handle_event():
   elif isinstance(body.event, NewEvent):
       return await handle_new_event(body.event, temporal_client)
   ```

### Testing

#### Unit Testing Activities

```python
import pytest
from pydantic_temporal_example.temporal.slack_activities import slack_conversations_replies
from pydantic_temporal_example.models import SlackConversationsRepliesRequest

@pytest.mark.asyncio
async def test_slack_conversations_replies():
    request = SlackConversationsRepliesRequest(
        channel="C12345",
        ts="1234567890.123456"
    )

    messages = await slack_conversations_replies(request)
    assert isinstance(messages, list)
    assert len(messages) > 0
```

#### Integration Testing Workflows

```python
from temporalio.testing import WorkflowEnvironment
from pydantic_temporal_example.temporal.workflows import SlackThreadWorkflow
from pydantic_temporal_example.models import AppMentionEvent

@pytest.mark.asyncio
async def test_workflow():
    async with await WorkflowEnvironment.start_time_skipping() as env:
        async with Worker(
            env.client,
            task_queue="test-queue",
            workflows=[SlackThreadWorkflow],
            activities=ALL_SLACK_ACTIVITIES
        ):
            event = AppMentionEvent(
                type="app_mention",
                user="U12345",
                text="@bot test",
                ts="1000.000",
                channel="C12345",
                event_ts="1000.000"
            )

            await env.client.start_workflow(
                SlackThreadWorkflow.run,
                id="test-workflow",
                start_signal="submit_app_mention_event",
                start_signal_args=[event],
                task_queue="test-queue"
            )
```

#### Testing Agents

```python
from pydantic_temporal_example.agents.dispatch_agent import dispatch_agent, SlackResponse
import json

@pytest.mark.asyncio
async def test_dispatcher():
    thread = json.dumps([
        {"user": "U12345", "text": "@bot I want Thai food"}
    ])

    result = await dispatch_agent.run(thread)
    assert isinstance(result.output, SlackResponse)
```

### Debugging

#### Enable Debug Logging

```python
# In app.py, add before logfire.configure()
import logging
logging.basicConfig(level=logging.DEBUG)
```

#### Inspect Workflow State

```python
from pydantic_temporal_example.temporal.client import build_temporal_client

client = await build_temporal_client()
handle = client.get_workflow_handle_for(
    SlackThreadWorkflow.run,
    workflow_id="app-mention-1234567890-123456"
)

# Get workflow info
info = await handle.describe()
print(f"Status: {info.status}")
print(f"Start time: {info.start_time}")

# Query workflow (if queries are defined)
# result = await handle.query("get_state")
```

#### View Temporal UI

```bash
# Start with UI
temporal server start-dev --ui-port 8233

# Navigate to http://localhost:8233
# View:
# - Workflow history
# - Activity executions
# - Signal history
# - Error stack traces
```

#### Logfire Debugging

```python
# View traces in Logfire dashboard
# https://logfire.pydantic.dev

# Logs include:
# - All agent executions with prompts/responses
# - HTTP requests to Slack/OpenAI
# - Activity executions
# - Workflow events
```

---

## References

### Detailed Documentation

| Document | Description | Path |
|----------|-------------|------|
| **Component Inventory** | Complete catalog of all modules, classes, functions with line numbers | [docs/01_component_inventory.md](./docs/01_component_inventory.md) |
| **Architecture Diagrams** | Mermaid diagrams showing system architecture, component relationships, data flow | [diagrams/02_architecture_diagrams.md](./diagrams/02_architecture_diagrams.md) |
| **Data Flow Analysis** | Sequence diagrams for all major flows, error handling patterns | [docs/03_data_flows.md](./docs/03_data_flows.md) |
| **API Reference** | Complete API documentation with examples, configuration options | [docs/04_api_reference.md](./docs/04_api_reference.md) |

### External Resources

**PydanticAI:**
- Documentation: https://ai.pydantic.dev/
- GitHub: https://github.com/pydantic/pydantic-ai
- Temporal Integration: https://ai.pydantic.dev/durable/

**Temporal:**
- Documentation: https://docs.temporal.io/
- Python SDK: https://docs.temporal.io/dev-guide/python
- Concepts: https://docs.temporal.io/concepts

**Slack API:**
- Events API: https://api.slack.com/apis/connections/events-api
- Block Kit: https://api.slack.com/block-kit
- Python SDK: https://slack.dev/python-slack-sdk/

**FastAPI:**
- Documentation: https://fastapi.tiangolo.com/
- Dependency Injection: https://fastapi.tiangolo.com/tutorial/dependencies/
- Lifespan Events: https://fastapi.tiangolo.com/advanced/events/

**Logfire:**
- Documentation: https://docs.pydantic.dev/logfire/
- Instrumentation: https://docs.pydantic.dev/logfire/integrations/

---

## Appendix

### Glossary

| Term | Definition |
|------|------------|
| **Activity** | Temporal's unit of external I/O - function that interacts with external services |
| **Agent** | PydanticAI construct representing an AI model with specific instructions and tools |
| **Block Kit** | Slack's UI framework for rich message formatting |
| **Discriminator** | Pydantic feature for type discrimination in unions based on a field value |
| **DuckDuckGo Tool** | PydanticAI tool that performs web searches via DuckDuckGo API |
| **HMAC** | Hash-based Message Authentication Code - used to verify webhook authenticity |
| **Logfire** | Pydantic's observability platform for logging and tracing |
| **Signal** | Temporal mechanism for sending data to a running workflow |
| **Temporal** | Workflow orchestration platform providing durable execution |
| **TemporalAgent** | Wrapper that allows PydanticAI agents to run in Temporal workflows |
| **Thread** | Slack conversation thread started by an original message |
| **Webhook** | HTTP callback from Slack when events occur |
| **Workflow** | Temporal's durable function that orchestrates activities and maintains state |

### Troubleshooting

#### Issue: "Invalid request signature" errors

**Cause:** HMAC signature verification failing

**Solutions:**
1. Verify `SLACK_SIGNING_SECRET` matches Slack app settings
2. Check that webhook URL is HTTPS (required by Slack)
3. Ensure clock sync between server and Slack (< 5 min difference)
4. Confirm request isn't being modified by proxy/load balancer

**Debug:**
```python
# In slack.py, add logging before signature check
print(f"Expected: {expected_signature}")
print(f"Received: {signature_header}")
print(f"Timestamp: {timestamp_header}")
print(f"Body: {request_body[:100]}")
```

#### Issue: Workflows not starting

**Cause:** Temporal server not running or not accessible

**Solutions:**
1. Check if Temporal server is running:
   ```bash
   temporal server start-dev
   ```
2. Verify `TEMPORAL_HOST` and `TEMPORAL_PORT` settings
3. Check worker logs for connection errors
4. Ensure task queue name matches between client and worker

**Debug:**
```python
# Test connection
from pydantic_temporal_example.temporal.client import build_temporal_client

client = await build_temporal_client()
await client.list_workflows()  # Should not error
```

#### Issue: Agent execution timeouts

**Cause:** OpenAI API slow or network issues

**Solutions:**
1. Increase activity timeout in workflow:
   ```python
   start_to_close_timeout=timedelta(seconds=30)  # Increase from 10
   ```
2. Check OpenAI API status: https://status.openai.com
3. Verify `OPENAI_API_KEY` is valid
4. Check network connectivity to api.openai.com

**Debug:**
```python
# Add timeout logging
import asyncio
import logfire

async def run_with_logging():
    start = asyncio.get_event_loop().time()
    result = await agent.run(input)
    duration = asyncio.get_event_loop().time() - start
    logfire.info(f"Agent execution took {duration:.2f}s")
    return result
```

#### Issue: Messages not appearing in Slack

**Cause:** Activity failures or workflow errors

**Solutions:**
1. Check Temporal UI for activity failures
2. Verify bot has permission to post in channel
3. Check Logfire for error traces
4. Ensure `SLACK_BOT_TOKEN` has `chat:write` scope

**Debug:**
```python
# In slack_activities.py, add detailed error logging
try:
    response = await client.chat_postMessage(...)
    logfire.info("Posted message", response=response.data)
except SlackApiError as e:
    logfire.error("Failed to post", error=str(e), response=e.response)
    raise
```

#### Issue: Duplicate messages or reactions

**Cause:** Activity retries after timeout

**Solutions:**
1. Make activities idempotent (Slack reactions are naturally idempotent)
2. Track posted message IDs to avoid duplicates
3. Reduce timeout values to fail faster
4. Use unique message IDs as workflow IDs

**Prevention:**
```python
# Store posted message ts in workflow state
self._posted_messages: set[str] = set()

# Before posting, check if already posted
if message_key not in self._posted_messages:
    result = await execute_activity(post_message, ...)
    self._posted_messages.add(message_key)
```

#### Issue: Bot ignoring messages

**Cause:** Workflow not found or bot filtering own messages

**Solutions:**
1. Ensure workflow ID is consistent (based on thread_ts)
2. Check that `slack_bot_user_id` is correct
3. Verify thread_ts is being extracted properly
4. Check logs for "No workflow found" messages

**Debug:**
```python
# In api.py, add logging
logfire.info(
    "Processing message",
    workflow_id=workflow_id,
    user=event.user,
    bot_id=slack_bot_user_id,
    thread_ts=event.thread_ts
)
```

---

## Architecture Patterns Summary

### Pattern 1: Event-Driven with Durable State

**Problem:** Need to maintain conversation state across multiple messages and server restarts.

**Solution:** Use Temporal workflows as long-running state machines, one per conversation thread.

**Benefits:**
- State survives crashes
- No external database needed
- Built-in retry logic
- Clear event history

### Pattern 2: Multi-Agent Delegation

**Problem:** Different requests need different capabilities and costs.

**Solution:** Use a fast dispatcher agent to route to specialized agents.

**Benefits:**
- Cost optimization (cheap agent for routing, expensive for research)
- Separation of concerns
- Easier to test and debug individual agents
- Can add new specialized agents without changing dispatcher

### Pattern 3: Activity-Based External I/O

**Problem:** External API calls can fail, timeout, or be rate-limited.

**Solution:** Isolate all external calls as Temporal activities with retry policies.

**Benefits:**
- Automatic retries on failure
- Timeout protection
- Independent testing
- Observability of each API call

### Pattern 4: Structured Agent Outputs

**Problem:** Parsing free-text agent responses is error-prone.

**Solution:** Use Pydantic dataclasses with discriminated unions for type-safe outputs.

**Benefits:**
- Compile-time and runtime type checking
- Self-documenting with field docstrings
- Pattern matching on output types
- Prevents parsing errors

### Pattern 5: Signal-Based Communication

**Problem:** Need to send data to running workflows without blocking.

**Solution:** Use Temporal signals to queue events in workflow.

**Benefits:**
- Non-blocking webhook handlers
- Workflows process events at their own pace
- Multiple signals can be queued
- Type-safe signal definitions

---

**For questions or contributions, refer to the detailed documentation and external resources listed above.**
