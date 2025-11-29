# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A production-ready Slack bot that recommends dinner options, demonstrating PydanticAI + Temporal integration patterns. The bot uses AI agents with structured outputs orchestrated through durable Temporal workflows.

**Key Technologies:** FastAPI, PydanticAI, Temporal, Slack SDK, DuckDuckGo search, Logfire observability

## Essential Documentation

**IMPORTANT:** This repository has extensive architecture documentation. Before making changes, consult the relevant docs:

### Architecture Documentation (`architecture/`)

| Document | Purpose | When to Use |
|----------|---------|-------------|
| [`architecture/README.md`](architecture/README.md) | High-level architecture overview, design decisions, quick start | First-time exploration, understanding the system |
| [`architecture/docs/01_component_inventory.md`](architecture/docs/01_component_inventory.md) | Complete catalog of all modules, classes, functions with line numbers | Finding specific code, understanding module purposes |
| [`architecture/diagrams/02_architecture_diagrams.md`](architecture/diagrams/02_architecture_diagrams.md) | Mermaid diagrams of system architecture, data flow, class hierarchies | Understanding component relationships, visual reference |
| [`architecture/docs/03_data_flows.md`](architecture/docs/03_data_flows.md) | Sequence diagrams for all major flows, error handling patterns | Debugging, tracing request lifecycles |
| [`architecture/docs/04_api_reference.md`](architecture/docs/04_api_reference.md) | Complete API documentation with examples, configuration options | Development, adding features, integration |

### Other Documentation

- `README.md` - Motivation, setup instructions, production considerations

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    External Systems                          │
│  Slack Platform │ OpenAI API │ DuckDuckGo │ Logfire         │
└─────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (FastAPI)                     │
│  /slack-events endpoint, HMAC verification, event routing    │
└─────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────┐
│            Temporal Orchestration Layer                      │
│  SlackThreadWorkflow (one per conversation thread)           │
│  Durable state, signals, event queue                        │
└─────────────────────────────────────────────────────────────┘
           ┌──────────────────┴──────────────────┐
┌──────────────────────┐              ┌─────────────────────┐
│   Activity Layer     │              │   Agent Layer       │
│  • slack_conversations_replies      │  • Dispatch agent   │
│  • slack_chat_post_message         │  • Research agent   │
│  • slack_reactions_add/remove      │  • DuckDuckGo tool  │
└──────────────────────┘              └─────────────────────┘
```

**Request Flow:**
1. Slack sends webhook → FastAPI validates HMAC signature
2. API creates/signals Temporal workflow (one per thread)
3. Workflow fetches thread messages, runs dispatch agent
4. Dispatch agent returns: `NoResponse`, `SlackResponse`, or `DinnerOptionResearchRequest`
5. If research needed → runs dinner research agent with DuckDuckGo
6. Response posted to Slack via activity

## Key Design Patterns

### 1. Multi-Agent Architecture
- **Dispatch Agent** (fast, GPT-4o-mini): Routes requests, asks clarifying questions
- **Research Agent** (capable, GPT-4o-mini): Web search, formats restaurant suggestions
- Cost optimization: cheap agent for routing, capable agent for research

### 2. Structured Agent Outputs with Discriminated Unions
```python
DispatchResult = NoResponse | SlackResponse | DinnerOptionResearchRequest
# Type-safe routing via pattern matching
```

### 3. Activity-Based External I/O
All Slack API calls isolated as Temporal activities with automatic retries and timeouts.

### 4. Thread-Based Durable Workflows
Each Slack thread = one workflow instance. State survives crashes, maintains conversation context.

## Commands

```bash
# Install dependencies
uv sync

# Format code (Ruff)
make format

# Type check (Pyright)
make typecheck

# Run the application (starts FastAPI + Temporal worker on port 4000)
uv run python -m pydantic_temporal_example.app
```

## File Organization

```
pydantic_temporal_example/
├── app.py                         # Entry point (FastAPI + Temporal startup)
├── api.py                         # /slack-events endpoint handler
├── models.py                      # Pydantic models for Slack events
├── settings.py                    # Configuration (env vars)
├── dependencies.py                # FastAPI dependency injection
├── slack.py                       # HMAC signature verification
├── agents/
│   ├── dispatch_agent.py          # Intent routing agent (lines 64-94)
│   └── dinner_research_agent.py   # Restaurant research agent (lines 35-43)
└── temporal/
    ├── workflows.py               # SlackThreadWorkflow (lines 35-108)
    ├── slack_activities.py        # Slack API activities (lines 12-81)
    ├── worker.py                  # Temporal worker setup (lines 17-36)
    └── client.py                  # Temporal client builder (lines 8-21)
```

## Adding New Features

**New Agent:**
1. Create agent in `agents/` with Pydantic dataclass outputs
2. Define: `Agent(model="openai-responses:gpt-4o-mini", output_type=..., instructions=..., tools=[...])`
3. Wrap for Temporal: `TemporalAgent(agent, name="...")`
4. Register in `worker.py` plugins, call from workflow

**New Slack Activity:**
1. Add function in `temporal/slack_activities.py` with `@activity.defn` and `@logfire.instrument`
2. Add to `ALL_SLACK_ACTIVITIES` list
3. Call from workflow: `await workflow.execute_activity(fn, args, start_to_close_timeout=timedelta(seconds=10))`

**New Event Type:**
1. Add Pydantic model in `models.py` with discriminator field
2. Update `SlackEvent` type alias
3. Add handler in `api.py`

See [`architecture/docs/04_api_reference.md`](architecture/docs/04_api_reference.md) for detailed examples.

## Environment Variables

**Required:**
- `SLACK_BOT_TOKEN` - Bot OAuth token (xoxb-...)
- `SLACK_SIGNING_SECRET` - For webhook verification
- `OPENAI_API_KEY` - For agent inference

**Optional:**
- `TEMPORAL_HOST` - Empty for local dev server
- `TEMPORAL_PORT` - Default 7233
- `TEMPORAL_TASK_QUEUE` - Default "agent-task-queue"
- Logfire token for observability

## Development Notes

- Python 3.12+ required
- Uses `uv` package manager
- Ruff for formatting (120 char lines)
- Pyright for type checking (strict mode)
- All external I/O isolated as Temporal activities for durability
- Thread context maintained in workflow state, survives crashes
- Comprehensive Logfire instrumentation throughout

## Troubleshooting

Common issues are documented in [`architecture/README.md`](architecture/README.md#troubleshooting):
- Invalid request signature errors
- Workflows not starting
- Agent execution timeouts
- Messages not appearing in Slack
- Duplicate messages or reactions
