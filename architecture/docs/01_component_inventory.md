# Component Inventory

## Overview

This project is a **Slack bot application** that uses **PydanticAI** agents and **Temporal workflows** to help users decide what to order for dinner. The codebase consists of 15 Python modules organized into a well-structured package (`pydantic_temporal_example`) with 4 sub-packages:

- **agents/** - PydanticAI agent definitions (dispatcher and dinner research agents)
- **temporal/** - Temporal workflow orchestration, activities, and client setup
- **Root package modules** - FastAPI application, API endpoints, models, settings, and Slack integration

The project follows a clean architecture with clear separation between:
- Public API components (agents, models, settings)
- Internal implementation (dependencies, Slack verification, Temporal internals)
- Entry points (FastAPI application with CLI main)

## Public API

### Modules

#### `pydantic_temporal_example/agents/dinner_research_agent.py`
**Purpose**: Defines the dinner research agent that performs web searches to find restaurant recommendations based on user preferences.

**Exports**:
- `DinnerOption` dataclass (lines 12-16)
- `DinnerSuggestions` dataclass (lines 19-32)
- `dinner_research_agent` agent instance (lines 35-43)

#### `pydantic_temporal_example/agents/dispatch_agent.py`
**Purpose**: Defines the dispatcher agent that determines user intent and routes requests appropriately.

**Exports**:
- `NoResponse` dataclass (lines 9-20)
- `SlackResponse` dataclass (lines 23-40)
- `DinnerOptionResearchRequest` dataclass (lines 43-59)
- `DispatchResult` type alias (line 62)
- `dispatch_agent` agent instance (lines 64-94)

#### `pydantic_temporal_example/models.py`
**Purpose**: Defines all Pydantic models for Slack events, messages, and API interactions.

**Exports**:
- `MessageChannelsEvent` class (lines 7-19) - Represents Slack channel message events
- `AppMentionEvent` class (lines 22-33) - Represents Slack app mention events
- `URLVerificationEvent` class (lines 36-39) - Handles Slack URL verification
- `SlackEvent` type alias (lines 42-44)
- `SlackEventsAPIBody` class (lines 47-55) - Main Slack events API payload
- `SlackEventsAPIBodyAdapter` type adapter (lines 58-60)
- `SlackMessageID` class (lines 63-65) - Identifies a Slack message
- `SlackReply` class (lines 68-78) - Represents a reply to be posted
- `SlackReaction` class (lines 81-83) - Represents a reaction to add/remove
- `SlackConversationsRepliesRequest` class (lines 86-91) - Request model for fetching thread replies

#### `pydantic_temporal_example/settings.py`
**Purpose**: Application settings management using pydantic-settings.

**Exports**:
- `Settings` class (lines 6-12) - Configuration model for the application
- `get_settings()` function (lines 15-17) - Cached settings getter

#### `pydantic_temporal_example/api.py`
**Purpose**: FastAPI router and endpoint handlers for Slack Events API.

**Exports**:
- `router` APIRouter instance (line 20)
- `handle_event()` async function (lines 23-48) - Main webhook endpoint for Slack events

### Classes

#### Agent Dataclasses (Public Output Types)

**DinnerOption** (`agents/dinner_research_agent.py:12-16`)
- **Purpose**: Represents a single restaurant option with details
- **Attributes**: `restaurant_name`, `restaurant_address`, `recommended_dishes`

**DinnerSuggestions** (`agents/dinner_research_agent.py:19-32`)
- **Purpose**: Output type for dinner research agent containing formatted suggestions
- **Attributes**: `response` (str or Slack blocks)

**NoResponse** (`agents/dispatch_agent.py:9-20`)
- **Purpose**: Indicates dispatcher determined no response is needed
- **Attributes**: `type` (literal "no-response")

**SlackResponse** (`agents/dispatch_agent.py:23-40`)
- **Purpose**: Immediate response from dispatcher without research
- **Attributes**: `type` (literal "slack-response"), `response` (str or Slack blocks)

**DinnerOptionResearchRequest** (`agents/dispatch_agent.py:43-59`)
- **Purpose**: Structured request to delegate to dinner research agent
- **Attributes**: `type`, `location`, `cuisine_preferences`, `price_preferences`, `dietary_restrictions`, `extra_info`

#### Slack Event Models (Public API Models)

**MessageChannelsEvent** (`models.py:7-19`)
- **Purpose**: Represents a message posted in a Slack channel
- **Properties**: `reply_thread_ts` (lines 17-19) - Returns thread timestamp for replies

**AppMentionEvent** (`models.py:22-33`)
- **Purpose**: Represents an @-mention of the bot in Slack
- **Properties**: `reply_thread_ts` (lines 31-33) - Returns thread timestamp for replies

**URLVerificationEvent** (`models.py:36-39`)
- **Purpose**: Handles Slack's URL verification challenge during setup

**SlackEventsAPIBody** (`models.py:47-55`)
- **Purpose**: Main container for Slack Events API webhook payloads

**SlackMessageID** (`models.py:63-65`)
- **Purpose**: Identifies a specific Slack message by channel and timestamp

**SlackReply** (`models.py:68-78`)
- **Purpose**: Represents a reply to be posted to Slack
- **Properties**: `text` (lines 72-74), `blocks` (lines 76-78)

**SlackReaction** (`models.py:81-83`)
- **Purpose**: Represents a reaction emoji to add/remove from a message

**SlackConversationsRepliesRequest** (`models.py:86-91`)
- **Purpose**: Request parameters for fetching thread replies from Slack API

#### Configuration

**Settings** (`settings.py:6-12`)
- **Purpose**: Application configuration using Pydantic BaseSettings
- **Attributes**: `slack_bot_token`, `slack_signing_secret`, `temporal_host`, `temporal_port`, `temporal_task_queue`

### Functions

#### Public API Functions

**get_settings()** (`settings.py:15-17`)
- **Purpose**: Returns cached application settings instance
- **Return Type**: `Settings`
- **Decorator**: `@cache` for singleton pattern

**handle_event()** (`api.py:23-48`)
- **Purpose**: Main FastAPI endpoint handler for Slack Events API webhooks
- **Dependencies**: `TemporalClient`, `slack_bot_user_id`, verified Slack event body
- **Return Type**: `Response`
- **Route**: POST `/slack-events`

#### Agent Instances

**dinner_research_agent** (`agents/dinner_research_agent.py:35-43`)
- **Purpose**: PydanticAI Agent configured for researching dinner options
- **Model**: `openai-responses:gpt-5-mini`
- **Output Type**: `DinnerSuggestions`
- **Tools**: DuckDuckGo search

**dispatch_agent** (`agents/dispatch_agent.py:64-94`)
- **Purpose**: PydanticAI Agent for routing and determining user intent
- **Model**: `openai-responses:gpt-5-mini`
- **Output Types**: `NoResponse | SlackResponse | DinnerOptionResearchRequest`
- **Tools**: DuckDuckGo search

## Internal Implementation

### Internal Modules

#### `pydantic_temporal_example/dependencies.py`
**Purpose**: Internal FastAPI dependency injection and lifecycle management.

**Contents**:
- `lifespan()` context manager (lines 14-20) - Manages application startup/shutdown
- `get_temporal_client()` dependency (lines 23-24) - Extracts Temporal client from request state
- `get_slack_bot_user_id()` dependency (lines 27-28) - Extracts Slack bot user ID from request state

#### `pydantic_temporal_example/slack.py`
**Purpose**: Internal Slack webhook verification logic.

**Contents**:
- `get_verified_slack_events_body()` async function (lines 13-48) - Verifies Slack webhook signatures and parses events

#### `pydantic_temporal_example/temporal/client.py`
**Purpose**: Internal Temporal client initialization and configuration.

**Contents**:
- `build_temporal_client()` async function (lines 8-21) - Constructs and configures Temporal client with plugins
- `_setup_logfire()` nested function (lines 13-17) - Configures Logfire instrumentation

#### `pydantic_temporal_example/temporal/slack_activities.py`
**Purpose**: Temporal activities for interacting with Slack API.

**Contents**:
- `slack_conversations_replies()` activity (lines 12-28) - Fetches thread messages with pagination
- `slack_chat_post_message()` activity (lines 31-40) - Posts a message to Slack
- `slack_chat_delete()` activity (lines 43-50) - Deletes a Slack message
- `slack_reactions_add()` activity (lines 53-61) - Adds a reaction emoji
- `slack_reactions_remove()` activity (lines 64-72) - Removes a reaction emoji
- `ALL_SLACK_ACTIVITIES` list (lines 75-81) - Registry of all Slack activities
- `_get_slack_client()` helper (lines 84-86) - Creates Slack API client

#### `pydantic_temporal_example/temporal/worker.py`
**Purpose**: Temporal worker initialization and lifecycle management.

**Contents**:
- `temporal_worker()` context manager (lines 17-36) - Sets up and runs Temporal worker with workflows, activities, and agent plugins

#### `pydantic_temporal_example/temporal/workflows.py`
**Purpose**: Temporal workflow definitions and agent orchestration.

**Contents**:
- `temporal_dispatch_agent` TemporalAgent instance (line 31)
- `temporal_dinner_research_agent` TemporalAgent instance (line 32)
- `SlackThreadWorkflow` workflow class (lines 35-107)
- `handle_user_request()` helper function (lines 110-118)

### Helper Classes

#### Workflow Class

**SlackThreadWorkflow** (`temporal/workflows.py:35-107`)
- **Purpose**: Temporal workflow that manages a single Slack thread conversation
- **State**:
  - `_pending_events` (line 38) - Queue of incoming Slack events
  - `_thread_messages` (line 39) - History of messages in the thread
- **Properties**:
  - `_most_recent_ts` (lines 41-46) - Returns timestamp of latest message
- **Methods**:
  - `run()` (lines 48-54) - Main workflow loop, processes events from queue
  - `submit_app_mention_event()` signal (lines 56-58) - Receives app mention events
  - `submit_message_channels_event()` signal (lines 60-62) - Receives channel message events
  - `handle_event()` (lines 64-107) - Processes a single event, orchestrates agents and Slack activities

### Utility Functions

#### Internal API Handlers

**handle_url_verification_event()** (`api.py:51-52`)
- **Purpose**: Responds to Slack's URL verification challenge
- **Parameters**: `URLVerificationEvent`
- **Returns**: `JSONResponse` with challenge value

**handle_app_mention_event()** (`api.py:55-65`)
- **Purpose**: Starts a Temporal workflow when bot is mentioned
- **Parameters**: `AppMentionEvent`, `TemporalClient`
- **Returns**: `Response` (204 No Content)
- **Implementation**: Creates workflow ID from thread timestamp, starts workflow with signal

**handle_message_channels_event()** (`api.py:68-78`)
- **Purpose**: Signals an existing workflow with new channel messages
- **Parameters**: `MessageChannelsEvent`, `TemporalClient`
- **Returns**: `Response` (204 No Content)
- **Implementation**: Gets workflow handle, describes it to check existence, sends signal if exists

#### Temporal Client Builder

**build_temporal_client()** (`temporal/client.py:8-21`)
- **Purpose**: Constructs Temporal client with PydanticAI and Logfire plugins
- **Returns**: `TemporalClient`
- **Configuration**: Connects to configured host:port, installs instrumentation plugins

#### Slack Verification

**get_verified_slack_events_body()** (`slack.py:13-48`)
- **Purpose**: Verifies Slack webhook signature and parses event body
- **Returns**: `SlackEventsAPIBody | URLVerificationEvent | dict[str, Any]`
- **Security**:
  - Validates timestamp (lines 18-25) - Rejects requests older than 5 minutes
  - Validates HMAC signature (lines 28-46) - Prevents request tampering
  - Uses constant-time comparison (line 45)

#### Agent Orchestration

**handle_user_request()** (`temporal/workflows.py:110-118`)
- **Purpose**: Orchestrates dispatcher and research agents to handle user request
- **Parameters**: `stringified_thread` (JSON string of thread messages)
- **Returns**: `NoResponse | SlackResponse | DinnerSuggestions`
- **Flow**:
  - Runs dispatcher agent (line 112)
  - If dispatcher returns NoResponse or SlackResponse, returns immediately (lines 113-114)
  - Otherwise runs dinner research agent with user preferences (lines 115-117)
  - Returns research agent output (line 118)

#### Slack Client Factory

**_get_slack_client()** (`temporal/slack_activities.py:84-86`)
- **Purpose**: Creates Slack AsyncWebClient for activities
- **Returns**: `AsyncWebClient`
- **Configuration**: Uses bot token from settings, 60-second timeout

#### Logfire Setup

**_setup_logfire()** (`temporal/client.py:13-17`)
- **Purpose**: Configures Logfire instrumentation for Temporal workers
- **Returns**: `logfire.Logfire` instance
- **Instrumentation**: PydanticAI and HTTPX tracing

#### Dependency Functions

**lifespan()** (`dependencies.py:14-20`)
- **Purpose**: FastAPI lifespan context manager for startup/shutdown
- **Yields**: Dictionary with `temporal_client` and `slack_bot_user_id`
- **Setup**: Creates Slack client, authenticates to get bot user ID, builds Temporal client

**get_temporal_client()** (`dependencies.py:23-24`)
- **Purpose**: FastAPI dependency to extract Temporal client from request state
- **Parameters**: `Request`
- **Returns**: `TemporalClient`

**get_slack_bot_user_id()** (`dependencies.py:27-28`)
- **Purpose**: FastAPI dependency to extract bot user ID from request state
- **Parameters**: `Request`
- **Returns**: `str` (Slack user ID)

## Entry Points

### CLI Entry Points

None defined in `pyproject.toml`. The application uses a `__main__` pattern for execution.

### Main Functions

#### **main()** (`app.py:18-22`)
- **Purpose**: Application entry point that starts Temporal worker and FastAPI server
- **Location**: `pydantic_temporal_example/app.py:18-22`
- **Implementation**:
  - Starts Temporal worker as async context manager (line 19)
  - Configures uvicorn to serve FastAPI app on port 4000 (line 20)
  - Runs server within worker context (line 22)

#### **\_\_main\_\_ block** (`app.py:25-28`)
- **Purpose**: Enables running the module directly with `python -m pydantic_temporal_example.app`
- **Location**: `pydantic_temporal_example/app.py:25-28`
- **Implementation**: Imports asyncio and runs `main()` function

### Package Initialization

#### **pydantic_temporal_example/\_\_init\_\_.py** (line 1)
- **Purpose**: Package root initialization
- **Contents**: Empty file (package marker only)

#### **pydantic_temporal_example/agents/\_\_init\_\_.py** (line 1)
- **Purpose**: Agents sub-package initialization
- **Contents**: Empty file (package marker only)

#### **pydantic_temporal_example/temporal/\_\_init\_\_.py** (line 1)
- **Purpose**: Temporal sub-package initialization
- **Contents**: Empty file (package marker only)

### Application Startup

#### **app module** (`app.py`)
- **FastAPI app instance**: Created at line 9 with lifespan management
- **Router registration**: API router included at line 10
- **Instrumentation** (lines 12-15):
  - Logfire configured with service name "app"
  - PydanticAI instrumentation enabled
  - HTTPX instrumentation enabled with full capture
  - FastAPI instrumentation enabled

#### **Recommended execution**:
```bash
uv run python -m pydantic_temporal_example.app
```

This starts:
1. Temporal worker (with optional local Temporal server for development)
2. FastAPI server on port 4000
3. All Logfire instrumentation
4. Workflow and activity registration
5. Agent plugin initialization

## Type Aliases and Constants

### Type Aliases

**SlackEvent** (`models.py:42-44`)
- **Definition**: `Annotated[AppMentionEvent | MessageChannelsEvent, Discriminator("type")]`
- **Purpose**: Discriminated union of Slack event types

**DispatchResult** (`agents/dispatch_agent.py:62`)
- **Definition**: `NoResponse | SlackResponse | DinnerOptionResearchRequest`
- **Purpose**: Union of all possible dispatcher agent outputs

### Constants

**ALL_SLACK_ACTIVITIES** (`temporal/slack_activities.py:75-81`)
- **Type**: `list` of activity functions
- **Purpose**: Registry of all Temporal activities for Slack API interactions
- **Contents**: All 5 Slack activity functions for worker registration

## Dependencies and Imports

### External Dependencies (from pyproject.toml)
- **pydantic-ai** (>=1.0.2) - AI agent framework
- **logfire[fastapi]** (>=4.3.4) - Observability and instrumentation
- **fastapi** (>=0.115.0) - Web framework
- **uvicorn** (>=0.30.0) - ASGI server
- **slack-sdk** (>=3.36.0) - Slack API client
- **pydantic-ai-slim[duckduckgo]** (>=1.0.2) - PydanticAI with search tools
- **claude-agent-sdk** (>=0.1.10) - Claude agent integration

### Internal Module Dependencies

**Dependency Graph**:
```
app.py
├── api.py
│   ├── dependencies.py
│   │   ├── settings.py
│   │   └── temporal/client.py
│   │       └── settings.py
│   ├── models.py
│   ├── settings.py
│   ├── slack.py
│   │   ├── models.py
│   │   └── settings.py
│   └── temporal/workflows.py
│       ├── agents/dinner_research_agent.py
│       ├── agents/dispatch_agent.py
│       ├── models.py
│       └── temporal/slack_activities.py
│           ├── models.py
│           └── settings.py
└── temporal/worker.py
    ├── settings.py
    ├── temporal/client.py
    ├── temporal/slack_activities.py
    └── temporal/workflows.py
```

## Summary Statistics

- **Total Python Files**: 15
- **Public API Modules**: 5 (agents/*, models.py, settings.py, api.py, app.py)
- **Internal Implementation Modules**: 6 (dependencies.py, slack.py, temporal/*)
- **Package Markers**: 3 (\_\_init\_\_.py files)
- **Total Classes**: 16 (11 dataclasses/models, 1 workflow, 1 settings class, 3 base models)
- **Total Functions**: 15 (2 public API, 13 internal)
- **Temporal Activities**: 5 (all Slack API interactions)
- **Temporal Workflows**: 1 (SlackThreadWorkflow)
- **PydanticAI Agents**: 2 (dispatcher, dinner research)
- **FastAPI Routes**: 1 (POST /slack-events)
- **Entry Points**: 1 (main() in app.py)

## Architecture Highlights

1. **Clean Separation**: Public API (agents, models) clearly separated from internal implementation (Temporal, Slack integration)

2. **Type Safety**: Heavy use of Pydantic models and dataclasses for type-safe data handling

3. **Dependency Injection**: FastAPI dependencies used for clean service injection

4. **Context Managers**: Proper lifecycle management using async context managers for Temporal worker and application lifespan

5. **Instrumentation**: Comprehensive Logfire instrumentation throughout the application

6. **Security**: HMAC signature verification for Slack webhooks with timing attack protection

7. **Durable Execution**: Temporal workflows provide fault tolerance and state persistence

8. **Multi-Agent Pattern**: Dispatcher routes to specialized research agent when needed

9. **Activity Pattern**: All external API calls (Slack) isolated in Temporal activities

10. **Signal-Driven**: Workflows react to Slack events via Temporal signals for real-time responsiveness
