# Architecture Diagrams

## Overview

This project is a Slack bot built with FastAPI, Temporal, and PydanticAI that helps users decide what to order for dinner. The architecture follows a layered approach with clear separation of concerns:

- **External Interfaces**: Slack Events API for receiving messages
- **API Layer**: FastAPI endpoints for handling Slack webhook events
- **Orchestration Layer**: Temporal workflows for durable execution
- **Activity Layer**: Temporal activities for Slack API interactions
- **Agent Layer**: PydanticAI agents for intelligent conversation and research

The system uses Temporal to manage long-running workflows that handle entire Slack conversation threads, with PydanticAI agents providing intelligent responses and restaurant research capabilities.

## System Architecture

### Description

The system architecture shows the layered structure of the application. When a user mentions the bot in Slack or replies to a thread:

1. Slack sends a webhook to the FastAPI endpoint (`/slack-events`)
2. The API layer validates the request and starts/signals a Temporal workflow
3. The workflow orchestrates the conversation, executing activities and agents
4. Slack activities interact with the Slack API to read messages and post responses
5. PydanticAI agents (dispatch and dinner research) provide intelligent processing
6. External services (OpenAI via PydanticAI, DuckDuckGo for search) are called as needed

```mermaid
flowchart TB
    subgraph External["External Systems"]
        Slack[Slack Platform]
        OpenAI[OpenAI API]
        DuckDuckGo[DuckDuckGo Search]
    end

    subgraph API["API Layer (FastAPI)"]
        SlackEvents["/slack-events endpoint<br/>api.py"]
        Validation[Request Validation<br/>slack.py]
        EventHandlers[Event Handlers<br/>handle_app_mention_event<br/>handle_message_channels_event]
    end

    subgraph Orchestration["Temporal Orchestration Layer"]
        TempClient[Temporal Client<br/>temporal/client.py]
        Workflows[SlackThreadWorkflow<br/>temporal/workflows.py]
        Worker[Temporal Worker<br/>temporal/worker.py]
    end

    subgraph Activity["Activity Layer"]
        SlackActivities[Slack Activities<br/>temporal/slack_activities.py]
        ConvReplies[slack_conversations_replies]
        PostMessage[slack_chat_post_message]
        Reactions[slack_reactions_add/remove]
    end

    subgraph Agents["AI Agent Layer (PydanticAI)"]
        DispatchAgent[Dispatch Agent<br/>agents/dispatch_agent.py]
        DinnerAgent[Dinner Research Agent<br/>agents/dinner_research_agent.py]
        TemporalAgents[TemporalAgent Wrappers<br/>workflows.py]
    end

    subgraph Data["Data Models"]
        Models[Pydantic Models<br/>models.py]
        Settings[Settings<br/>settings.py]
    end

    subgraph Infrastructure["Infrastructure"]
        Deps[Dependencies<br/>dependencies.py]
        App[Application Bootstrap<br/>app.py]
        Logfire[Logfire Observability]
    end

    Slack -->|Webhook Events| SlackEvents
    SlackEvents --> Validation
    Validation --> EventHandlers
    EventHandlers -->|Start/Signal Workflow| TempClient
    TempClient --> Workflows
    Workflows --> SlackActivities
    Workflows --> TemporalAgents
    TemporalAgents --> DispatchAgent
    TemporalAgents --> DinnerAgent
    SlackActivities -->|API Calls| Slack
    DispatchAgent -->|Search| DuckDuckGo
    DinnerAgent -->|Search| DuckDuckGo
    DispatchAgent -->|LLM Calls| OpenAI
    DinnerAgent -->|LLM Calls| OpenAI
    Worker -.->|Executes| Workflows
    Worker -.->|Executes| SlackActivities
    Worker -.->|Executes| TemporalAgents
    App -.->|Starts| Worker
    App -.->|Creates| TempClient
    Deps -.->|Provides| TempClient
    Models -.->|Used by| API
    Models -.->|Used by| Workflows
    Settings -.->|Configures| Infrastructure
    Logfire -.->|Monitors| API
    Logfire -.->|Monitors| Agents
    Logfire -.->|Monitors| Workflows

    style External fill:#e1f5ff
    style API fill:#fff4e1
    style Orchestration fill:#ffe1f5
    style Activity fill:#e1ffe1
    style Agents fill:#f5e1ff
    style Data fill:#ffe1e1
    style Infrastructure fill:#f0f0f0
```

## Component Relationships

### Description

This diagram shows how the main Python modules connect and interact with each other. The arrows indicate import dependencies and data flow:

- `app.py` is the entry point that bootstraps the FastAPI application and Temporal worker
- `api.py` contains the route handlers that receive Slack events
- `dependencies.py` provides dependency injection for FastAPI
- `temporal/workflows.py` orchestrates the conversation flow
- `temporal/slack_activities.py` handles all Slack API interactions
- The agent modules define the AI agents with their instructions and tools

```mermaid
flowchart LR
    subgraph Main
        App[app.py<br/>FastAPI App<br/>+ Main Entry]
    end

    subgraph API
        ApiRoutes[api.py<br/>Slack Event Routes]
        Slack[slack.py<br/>Request Validation]
    end

    subgraph Config
        Settings[settings.py<br/>Configuration]
        Deps[dependencies.py<br/>DI Container]
        Models[models.py<br/>Data Models]
    end

    subgraph Temporal
        TClient[temporal/client.py<br/>Client Builder]
        TWorker[temporal/worker.py<br/>Worker Manager]
        TWorkflows[temporal/workflows.py<br/>SlackThreadWorkflow]
        TActivities[temporal/slack_activities.py<br/>Slack API Activities]
    end

    subgraph Agents
        Dispatch[agents/dispatch_agent.py<br/>Dispatch Agent]
        Dinner[agents/dinner_research_agent.py<br/>Dinner Research Agent]
    end

    App -->|includes router| ApiRoutes
    App -->|uses lifespan| Deps
    App -->|starts| TWorker
    ApiRoutes -->|validates| Slack
    ApiRoutes -->|imports models| Models
    ApiRoutes -->|gets client| Deps
    ApiRoutes -->|starts workflow| TWorkflows
    Deps -->|builds client| TClient
    Deps -->|reads config| Settings
    Slack -->|validates| Models
    Slack -->|reads config| Settings
    TWorker -->|builds client| TClient
    TWorker -->|registers| TWorkflows
    TWorker -->|registers| TActivities
    TWorker -->|registers agents| Dispatch
    TWorker -->|registers agents| Dinner
    TWorker -->|reads config| Settings
    TClient -->|reads config| Settings
    TWorkflows -->|imports models| Models
    TWorkflows -->|calls| TActivities
    TWorkflows -->|executes| Dispatch
    TWorkflows -->|executes| Dinner
    TActivities -->|imports models| Models
    TActivities -->|reads config| Settings

    style Main fill:#4a90e2
    style API fill:#e8a87c
    style Config fill:#c06c84
    style Temporal fill:#6c5ce7
    style Agents fill:#00b894
```

## Class Hierarchies

### Description

This diagram shows the class structure and relationships. The project uses Pydantic BaseModel extensively for data validation and dataclasses for agent output types. Key class categories:

- **Event Models**: Represent Slack events (AppMentionEvent, MessageChannelsEvent)
- **Message Models**: Represent Slack messages and interactions (SlackReply, SlackReaction, SlackMessageID)
- **Agent Output Types**: Define structured outputs from AI agents (NoResponse, SlackResponse, DinnerSuggestions)
- **Configuration**: Settings class for environment variables
- **Workflows**: SlackThreadWorkflow class that orchestrates conversation handling

```mermaid
---
id: febcc42b-3bdf-4e2b-a4d0-948d99439c41
---
classDiagram
    class BaseModel {
        <<Pydantic>>
    }

    class Settings {
        +slack_bot_token: str
        +slack_signing_secret: str
        +temporal_host: str | None
        +temporal_port: int
        +temporal_task_queue: str
    }

    class MessageChannelsEvent {
        +type: "message"
        +user: str
        +text: str
        +ts: str
        +channel: str
        +event_ts: str
        +channel_type: str
        +thread_ts: str | None
        +reply_thread_ts() str
    }

    class AppMentionEvent {
        +type: "app_mention"
        +user: str
        +text: str
        +ts: str
        +channel: str
        +event_ts: str
        +thread_ts: str | None
        +reply_thread_ts() str
    }

    class URLVerificationEvent {
        +type: "url_verification"
        +token: str
        +challenge: str
    }

    class SlackEventsAPIBody {
        +token: str
        +team_id: str
        +api_app_id: str
        +event: SlackEvent
        +type: "event_callback"
        +event_id: str
        +event_time: int
        +authed_users: list[str] | None
    }

    class SlackMessageID {
        +channel: str
        +ts: str
    }

    class SlackReply {
        +thread: SlackMessageID
        +content: str | list[dict]
        +text() str | None
        +blocks() list[dict] | None
    }

    class SlackReaction {
        +message: SlackMessageID
        +name: str
    }

    class SlackConversationsRepliesRequest {
        +channel: str
        +ts: str
        +oldest: str | None
    }

    class NoResponse {
        <<dataclass>>
        +type: "no-response"
    }

    class SlackResponse {
        <<dataclass>>
        +type: "slack-response"
        +response: str | list[dict]
    }

    class DinnerOptionResearchRequest {
        <<dataclass>>
        +type: "dinner-option-research-request"
        +location: WebSearchUserLocation
        +cuisine_preferences: str
        +price_preferences: str
        +dietary_restrictions: str
        +extra_info: str | None
    }

    class DinnerOption {
        <<dataclass>>
        +restaurant_name: str
        +restaurant_address: str | None
        +recommended_dishes: str
    }

    class DinnerSuggestions {
        <<dataclass>>
        +response: str | list[dict]
    }

    class SlackThreadWorkflow {
        <<Temporal Workflow>>
        -_pending_events: Queue
        -_thread_messages: list[dict]
        -_most_recent_ts: str | None
        +run() None
        +submit_app_mention_event(event) None
        +submit_message_channels_event(event) None
        +handle_event(event) None
    }

    class Agent {
        <<PydanticAI>>
        +model: str
        +output_type: type
        +instructions: str
        +tools: list
    }

    class TemporalAgent {
        <<PydanticAI>>
        +agent: Agent
        +name: str
    }

    BaseModel <|-- Settings
    BaseModel <|-- MessageChannelsEvent
    BaseModel <|-- AppMentionEvent
    BaseModel <|-- URLVerificationEvent
    BaseModel <|-- SlackEventsAPIBody
    BaseModel <|-- SlackMessageID
    BaseModel <|-- SlackReply
    BaseModel <|-- SlackReaction
    BaseModel <|-- SlackConversationsRepliesRequest

    SlackEventsAPIBody --> MessageChannelsEvent
    SlackEventsAPIBody --> AppMentionEvent
    SlackReply --> SlackMessageID
    SlackReaction --> SlackMessageID

    Agent <.. TemporalAgent : wraps

    SlackThreadWorkflow ..> AppMentionEvent : processes
    SlackThreadWorkflow ..> MessageChannelsEvent : processes
    SlackThreadWorkflow ..> TemporalAgent : executes

    NoResponse ..> Agent : output
    SlackResponse ..> Agent : output
    DinnerOptionResearchRequest ..> Agent : output
    DinnerSuggestions ..> Agent : output
```

## Module Dependencies

### Description

This diagram shows the import dependencies between Python modules in the project. Arrows point from modules that import to modules being imported. The structure follows a clean dependency hierarchy:

- Core configuration modules (settings, models) have no internal dependencies
- Infrastructure modules (dependencies, slack) depend on configuration
- API layer depends on models, settings, and Temporal
- Temporal layer depends on configuration and agents
- Application entry point depends on all layers

```mermaid
flowchart TD
    subgraph Core["Core Modules"]
        Settings[settings.py<br/>Configuration]
        Models[models.py<br/>Data Models]
    end

    subgraph Infrastructure["Infrastructure"]
        Slack[slack.py<br/>Slack Validation]
        Deps[dependencies.py<br/>Dependency Injection]
    end

    subgraph API["API Layer"]
        ApiPy[api.py<br/>Route Handlers]
    end

    subgraph Temporal["Temporal Layer"]
        TClient[temporal/client.py<br/>Client]
        TWorker[temporal/worker.py<br/>Worker]
        TWorkflows[temporal/workflows.py<br/>Workflows]
        TActivities[temporal/slack_activities.py<br/>Activities]
    end

    subgraph Agents["Agent Layer"]
        DispatchAgent[agents/dispatch_agent.py<br/>Dispatch Agent]
        DinnerAgent[agents/dinner_research_agent.py<br/>Dinner Agent]
    end

    subgraph Entry["Entry Point"]
        App[app.py<br/>Application]
    end

    Slack --> Settings
    Slack --> Models
    Deps --> Settings
    Deps --> TClient
    ApiPy --> Models
    ApiPy --> Settings
    ApiPy --> Slack
    ApiPy --> Deps
    ApiPy --> TWorkflows
    TClient --> Settings
    TActivities --> Settings
    TActivities --> Models
    TWorkflows --> Models
    TWorkflows --> TActivities
    TWorkflows --> DispatchAgent
    TWorkflows --> DinnerAgent
    TWorker --> Settings
    TWorker --> TClient
    TWorker --> TWorkflows
    TWorker --> TActivities
    TWorker --> DispatchAgent
    TWorker --> DinnerAgent
    App --> ApiPy
    App --> Deps
    App --> TWorker

    style Core fill:#ffeb99
    style Infrastructure fill:#c7e9ff
    style API fill:#ffcccc
    style Temporal fill:#e6ccff
    style Agents fill:#ccffcc
    style Entry fill:#ffd9b3
```

## Data Flow

### Description

This sequence diagram shows how data flows through the system when a user mentions the bot in Slack. The flow demonstrates:

1. Slack webhook event delivery
2. Event validation and workflow creation
3. Workflow orchestration of activities and agents
4. Message retrieval from Slack
5. AI agent processing (dispatch then dinner research)
6. Response posting back to Slack

The workflow maintains state across the entire conversation thread and coordinates all asynchronous operations.

```mermaid
sequenceDiagram
    participant User as Slack User
    participant Slack as Slack Platform
    participant API as FastAPI<br/>(api.py)
    participant Valid as Request Validator<br/>(slack.py)
    participant Client as Temporal Client
    participant Workflow as SlackThreadWorkflow
    participant Activities as Slack Activities
    participant DispatchAgent as Dispatch Agent
    participant DinnerAgent as Dinner Research Agent
    participant DuckDuckGo as DuckDuckGo
    participant OpenAI as OpenAI API

    User->>Slack: Mentions bot in message
    Slack->>API: POST /slack-events<br/>(app_mention event)
    API->>Valid: Validate signature
    Valid->>Valid: Verify HMAC signature
    Valid->>Valid: Parse event body
    Valid-->>API: AppMentionEvent
    API->>Client: Start workflow<br/>(with start signal)
    Client->>Workflow: Create instance
    Client->>Workflow: Signal: submit_app_mention_event
    API-->>Slack: 204 No Content

    Workflow->>Workflow: Queue event
    Workflow->>Workflow: Process event
    Workflow->>Activities: slack_reactions_add<br/>(add spinner)
    Activities->>Slack: reactions.add API
    Slack-->>Activities: OK

    Workflow->>Activities: slack_conversations_replies<br/>(get thread messages)
    Activities->>Slack: conversations.replies API
    Slack-->>Activities: Thread messages
    Activities-->>Workflow: Messages list

    Workflow->>Workflow: Update thread state
    Workflow->>DispatchAgent: Run dispatch agent<br/>(with thread context)
    DispatchAgent->>OpenAI: LLM request<br/>(analyze thread)
    OpenAI-->>DispatchAgent: Analysis

    alt User info complete
        DispatchAgent-->>Workflow: DinnerOptionResearchRequest
        Workflow->>DinnerAgent: Run dinner research<br/>(with preferences)
        DinnerAgent->>DuckDuckGo: Search restaurants
        DuckDuckGo-->>DinnerAgent: Search results
        DinnerAgent->>OpenAI: LLM request<br/>(format suggestions)
        OpenAI-->>DinnerAgent: Formatted response
        DinnerAgent-->>Workflow: DinnerSuggestions
    else Need more info
        DispatchAgent-->>Workflow: SlackResponse<br/>(clarifying question)
    else No response needed
        DispatchAgent-->>Workflow: NoResponse
    end

    alt Response to send
        par Post response and remove spinner
            Workflow->>Activities: slack_reactions_remove<br/>(remove spinner)
            Activities->>Slack: reactions.remove API
            and
            Workflow->>Activities: slack_chat_post_message
            Activities->>Slack: chat.postMessage API<br/>(in thread)
            Slack->>User: Shows message
        end
    end

    Note over Workflow: Workflow continues running,<br/>waiting for next event

    User->>Slack: Replies in thread
    Slack->>API: POST /slack-events<br/>(message event)
    API->>Client: Signal existing workflow
    Client->>Workflow: Signal: submit_message_channels_event
    Note over Workflow: Process repeats...
```

## External Service Integration

### Description

This diagram shows how the system integrates with external services and APIs:

- **Slack API**: For receiving events and posting messages
- **Temporal Server**: For workflow orchestration (local or cloud)
- **OpenAI API**: For LLM capabilities via PydanticAI
- **DuckDuckGo**: For web search via PydanticAI tools
- **Logfire**: For observability and monitoring

```mermaid
flowchart TB
    subgraph Application["Pydantic Temporal Example Application"]
        API[FastAPI API]
        Worker[Temporal Worker]
        Agents[PydanticAI Agents]
    end

    subgraph External["External Services"]
        SlackAPI[Slack API<br/>slack.com]
        TemporalServer[Temporal Server<br/>localhost:7233 or cloud]
        OpenAIAPI[OpenAI API<br/>api.openai.com]
        DDGAPI[DuckDuckGo Search]
        LogfireService[Logfire<br/>logfire.pydantic.dev]
    end

    SlackAPI -->|Webhooks<br/>Events API| API
    API -->|Post messages<br/>Add reactions<br/>Get thread history| SlackAPI

    API -->|Start workflows<br/>Send signals| TemporalServer
    Worker -->|Register & execute<br/>workflows/activities| TemporalServer

    Agents -->|Chat completions<br/>gpt-5-mini| OpenAIAPI
    Agents -->|Web search<br/>queries| DDGAPI

    API -.->|Send logs/traces| LogfireService
    Worker -.->|Send logs/traces| LogfireService
    Agents -.->|Send logs/traces| LogfireService

    style Application fill:#e3f2fd
    style External fill:#fff3e0
```

## Key Architectural Patterns

### 1. Event-Driven Architecture
- Slack events trigger workflow creation/signaling
- Workflows queue events and process them asynchronously
- Activities interact with external services

### 2. Durable Execution
- Temporal ensures workflows survive crashes and restarts
- Conversation state is maintained across long-running threads
- Activities are retryable and fault-tolerant

### 3. Agent-Based AI
- PydanticAI agents provide structured outputs with type safety
- Dispatch agent determines intent and routes to specialized agents
- Dinner research agent performs specialized research tasks

### 4. Dependency Injection
- FastAPI dependencies provide Temporal client and Slack configuration
- Settings are centralized and cached
- Lifespan context manages application lifecycle

### 5. Type Safety
- Pydantic models validate all data at runtime
- Type hints throughout codebase
- Pyright strict mode enabled

## Technology Stack

| Layer | Technologies |
|-------|-------------|
| Web Framework | FastAPI 0.115.0+ |
| AI Framework | PydanticAI 1.0.2+ |
| Workflow Engine | Temporal (via temporalio) |
| Data Validation | Pydantic |
| External APIs | Slack SDK 3.36.0+ |
| LLM Provider | OpenAI (gpt-5-mini) |
| Search | DuckDuckGo |
| Observability | Logfire 4.3.4+ |
| Language | Python 3.12+ |

## File Reference

### Core Application Files
- `pydantic_temporal_example/app.py` - Application entry point
- `pydantic_temporal_example/api.py` - FastAPI routes (lines 23-78)
- `pydantic_temporal_example/models.py` - Pydantic data models (lines 1-92)
- `pydantic_temporal_example/settings.py` - Configuration (lines 6-12)

### Temporal Components
- `pydantic_temporal_example/temporal/workflows.py` - SlackThreadWorkflow (lines 35-108)
- `pydantic_temporal_example/temporal/slack_activities.py` - Slack API activities (lines 12-82)
- `pydantic_temporal_example/temporal/worker.py` - Worker setup (lines 17-36)
- `pydantic_temporal_example/temporal/client.py` - Client builder (lines 8-21)

### AI Agents
- `pydantic_temporal_example/agents/dispatch_agent.py` - Intent classification (lines 64-94)
- `pydantic_temporal_example/agents/dinner_research_agent.py` - Restaurant research (lines 35-43)

### Infrastructure
- `pydantic_temporal_example/dependencies.py` - FastAPI dependencies (lines 14-28)
- `pydantic_temporal_example/slack.py` - Request validation (lines 13-48)
