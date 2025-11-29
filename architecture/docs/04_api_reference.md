# API Reference

## Overview

This document provides a comprehensive reference for all public functions, classes, models, and configuration options in the PydanticAI Temporal Example project. This project demonstrates how to build durable AI agents using PydanticAI and Temporal for a Slack bot that helps users decide what to order for dinner.

The API is organized into several key areas:
- **Configuration**: Environment variables and settings management
- **Data Models**: Pydantic models for Slack events, messages, and agent outputs
- **API Endpoints**: FastAPI endpoints for handling Slack events
- **Temporal Components**: Workflows and activities for durable execution
- **AI Agents**: Dispatcher and research agents with structured outputs
- **Utilities**: Helper functions for Slack integration and Temporal client management

## Table of Contents

- [Configuration](#configuration)
  - [Environment Variables](#environment-variables)
  - [Settings Class](#settings-class)
- [Data Models](#data-models)
  - [Slack Event Models](#slack-event-models)
  - [Slack Message Models](#slack-message-models)
  - [Agent Response Models](#agent-response-models)
- [API Endpoints](#api-endpoints)
  - [POST /slack-events](#post-slack-events)
- [Temporal Components](#temporal-components)
  - [Workflows](#workflows)
  - [Activities](#activities)
  - [Temporal Client](#temporal-client)
  - [Temporal Worker](#temporal-worker)
- [AI Agents](#ai-agents)
  - [Dispatcher Agent](#dispatcher-agent)
  - [Dinner Research Agent](#dinner-research-agent)
- [Utility Functions](#utility-functions)
  - [Slack Utilities](#slack-utilities)
  - [Dependencies](#dependencies)
- [Usage Examples](#usage-examples)
  - [Basic Usage](#basic-usage)
  - [Advanced Patterns](#advanced-patterns)
- [Best Practices](#best-practices)
- [Error Handling](#error-handling)

---

## Configuration

### Environment Variables

The application requires the following environment variables to be configured:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `SLACK_BOT_TOKEN` | str | Yes | None | OAuth token for the Slack bot (starts with `xoxb-`) |
| `SLACK_SIGNING_SECRET` | str | Yes | None | Signing secret for verifying Slack webhook requests |
| `TEMPORAL_HOST` | str | No | None | Temporal server host address. `None` starts a local instance |
| `TEMPORAL_PORT` | int | No | 7233 | Port for Temporal server connection |
| `TEMPORAL_TASK_QUEUE` | str | No | "agent-task-queue" | Task queue name for Temporal workers |
| `OPENAI_API_KEY` | str | Yes | None | OpenAI API key for running AI agents |

**Source:** `pydantic_temporal_example/settings.py` (lines 6-12)

### Settings Class

#### `Settings`

Configuration settings loaded from environment variables using Pydantic Settings.

**Source:** `pydantic_temporal_example/settings.py` (lines 6-13)

```python
class Settings(BaseSettings):
    slack_bot_token: str
    slack_signing_secret: str
    temporal_host: str | None = None
    temporal_port: int = 7233
    temporal_task_queue: str = "agent-task-queue"
```

**Attributes:**
- `slack_bot_token` (str): Slack bot OAuth token
- `slack_signing_secret` (str): Secret for verifying Slack requests
- `temporal_host` (str | None): Temporal server host; `None` starts local dev instance
- `temporal_port` (int): Temporal server port (default: 7233)
- `temporal_task_queue` (str): Task queue name (default: "agent-task-queue")

**Example:**

```python
from pydantic_temporal_example.settings import Settings

# Loads from environment variables
settings = Settings()
print(settings.temporal_task_queue)  # "agent-task-queue"
```

#### `get_settings()`

Returns a cached singleton instance of Settings.

**Source:** `pydantic_temporal_example/settings.py` (lines 15-17)

**Signature:**
```python
@cache
def get_settings() -> Settings
```

**Returns:** `Settings` - The application settings instance

**Example:**

```python
from pydantic_temporal_example.settings import get_settings

settings = get_settings()
# Always returns the same instance (cached)
```

---

## Data Models

### Slack Event Models

#### `MessageChannelsEvent`

Represents a message event from Slack channels.

**Source:** `pydantic_temporal_example/models.py` (lines 7-20)

```python
class MessageChannelsEvent(BaseModel):
    type: Literal["message"]
    user: str
    text: str
    ts: str
    channel: str
    event_ts: str
    channel_type: str
    thread_ts: str | None = None
```

**Fields:**
- `type`: Always "message" (discriminator field)
- `user`: Slack user ID who sent the message
- `text`: Message text content
- `ts`: Timestamp of the message (used as message ID)
- `channel`: Channel ID where message was sent
- `event_ts`: Event timestamp
- `channel_type`: Type of channel (e.g., "channel", "im")
- `thread_ts`: Thread timestamp if message is in a thread (optional)

**Properties:**
- `reply_thread_ts` (str): Returns the thread timestamp or message timestamp

**Example:**

```python
from pydantic_temporal_example.models import MessageChannelsEvent

event = MessageChannelsEvent(
    type="message",
    user="U12345",
    text="What should I eat for dinner?",
    ts="1234567890.123456",
    channel="C12345",
    event_ts="1234567890.123456",
    channel_type="channel"
)

# Get thread timestamp for reply
reply_ts = event.reply_thread_ts  # "1234567890.123456"
```

#### `AppMentionEvent`

Represents an app mention event when the bot is @mentioned.

**Source:** `pydantic_temporal_example/models.py` (lines 22-34)

```python
class AppMentionEvent(BaseModel):
    type: Literal["app_mention"]
    user: str
    text: str
    ts: str
    channel: str
    event_ts: str
    thread_ts: str | None = None
```

**Fields:**
- `type`: Always "app_mention" (discriminator field)
- `user`: Slack user ID who mentioned the bot
- `text`: Message text including the mention
- `ts`: Timestamp of the message
- `channel`: Channel ID where mention occurred
- `event_ts`: Event timestamp
- `thread_ts`: Thread timestamp if mention is in a thread (optional)

**Properties:**
- `reply_thread_ts` (str): Returns the thread timestamp or message timestamp

**Example:**

```python
from pydantic_temporal_example.models import AppMentionEvent

event = AppMentionEvent(
    type="app_mention",
    user="U12345",
    text="@dinnerbot Help me find Italian food",
    ts="1234567890.123456",
    channel="C12345",
    event_ts="1234567890.123456"
)
```

#### `URLVerificationEvent`

Represents a URL verification challenge from Slack.

**Source:** `pydantic_temporal_example/models.py` (lines 36-40)

```python
class URLVerificationEvent(BaseModel):
    type: Literal["url_verification"]
    token: str
    challenge: str
```

**Fields:**
- `type`: Always "url_verification"
- `token`: Verification token from Slack
- `challenge`: Challenge string to echo back

**Example:**

```python
from pydantic_temporal_example.models import URLVerificationEvent

event = URLVerificationEvent(
    type="url_verification",
    token="abc123",
    challenge="xyz789"
)
```

#### `SlackEventsAPIBody`

Wrapper for Slack Events API callback payloads.

**Source:** `pydantic_temporal_example/models.py` (lines 47-56)

```python
class SlackEventsAPIBody(BaseModel):
    token: str
    team_id: str
    api_app_id: str
    event: SlackEvent
    type: Literal["event_callback"]
    event_id: str
    event_time: int
    authed_users: list[str] | None = None
```

**Fields:**
- `token`: Verification token
- `team_id`: Slack team/workspace ID
- `api_app_id`: Slack app ID
- `event`: The actual event (AppMentionEvent or MessageChannelsEvent)
- `type`: Always "event_callback"
- `event_id`: Unique event ID
- `event_time`: Unix timestamp
- `authed_users`: List of authorized user IDs (optional)

**Example:**

```python
from pydantic_temporal_example.models import SlackEventsAPIBody, AppMentionEvent

body = SlackEventsAPIBody(
    token="token123",
    team_id="T12345",
    api_app_id="A12345",
    event=AppMentionEvent(
        type="app_mention",
        user="U12345",
        text="@bot hello",
        ts="1234567890.123456",
        channel="C12345",
        event_ts="1234567890.123456"
    ),
    type="event_callback",
    event_id="Ev12345",
    event_time=1234567890
)
```

### Slack Message Models

#### `SlackMessageID`

Identifies a Slack message by channel and timestamp.

**Source:** `pydantic_temporal_example/models.py` (lines 63-66)

```python
class SlackMessageID(BaseModel):
    channel: str
    ts: str
```

**Fields:**
- `channel`: Channel ID
- `ts`: Message timestamp (serves as unique ID within channel)

**Example:**

```python
from pydantic_temporal_example.models import SlackMessageID

msg_id = SlackMessageID(channel="C12345", ts="1234567890.123456")
```

#### `SlackReply`

Represents a reply to post to a Slack thread.

**Source:** `pydantic_temporal_example/models.py` (lines 68-79)

```python
class SlackReply(BaseModel):
    thread: SlackMessageID
    content: str | list[dict[str, Any]]
```

**Fields:**
- `thread`: The thread to reply to (identified by channel and timestamp)
- `content`: Either markdown text string or Slack Block Kit blocks

**Properties:**
- `text` (str | None): Returns content if it's a string, else None
- `blocks` (list[dict[str, Any]] | None): Returns content if it's blocks, else None

**Example:**

```python
from pydantic_temporal_example.models import SlackReply, SlackMessageID

# Text reply
reply = SlackReply(
    thread=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    content="Here are some dinner suggestions!"
)

# Block Kit reply
reply = SlackReply(
    thread=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    content=[
        {
            "type": "section",
            "text": {"type": "mrkdwn", "text": "*Restaurant Name*"}
        }
    ]
)
```

#### `SlackReaction`

Represents a reaction (emoji) to add/remove from a Slack message.

**Source:** `pydantic_temporal_example/models.py` (lines 81-84)

```python
class SlackReaction(BaseModel):
    message: SlackMessageID
    name: str
```

**Fields:**
- `message`: The message to react to
- `name`: Emoji name without colons (e.g., "thumbsup", "spin")

**Example:**

```python
from pydantic_temporal_example.models import SlackReaction, SlackMessageID

reaction = SlackReaction(
    message=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    name="spin"  # Shows :spin: emoji (thinking indicator)
)
```

#### `SlackConversationsRepliesRequest`

Request parameters for fetching thread messages.

**Source:** `pydantic_temporal_example/models.py` (lines 86-92)

```python
class SlackConversationsRepliesRequest(BaseModel):
    channel: str
    ts: str
    oldest: str | None
```

**Fields:**
- `channel`: Channel ID
- `ts`: Thread timestamp
- `oldest`: Only include messages after this timestamp (optional)

**Example:**

```python
from pydantic_temporal_example.models import SlackConversationsRepliesRequest

request = SlackConversationsRepliesRequest(
    channel="C12345",
    ts="1234567890.123456",
    oldest="1234567800.000000"  # Only get messages after this
)
```

### Agent Response Models

#### `NoResponse`

Indicates the agent should not respond to the current message.

**Source:** `pydantic_temporal_example/agents/dispatch_agent.py` (lines 9-21)

```python
@dataclass
@with_config(use_attribute_docstrings=True)
class NoResponse:
    """A marker that indicates that you do not currently need to reply to the thread.

    You should use this when the most recent messages in the thread are not directed
    toward you, e.g. if other users are discussing something between themselves.
    """
    type: Literal["no-response"]
```

**Fields:**
- `type`: Always "no-response" (discriminator)

**Use Case:** When users are talking among themselves in a thread where the bot was previously mentioned.

#### `SlackResponse`

Indicates the agent should send an immediate response without research.

**Source:** `pydantic_temporal_example/agents/dispatch_agent.py` (lines 23-41)

```python
@dataclass
@with_config(use_attribute_docstrings=True)
class SlackResponse:
    """A marker that indicates that you want to immediately send a response message.

    You can use this to request additional information from the user if they have
    not provided all the information necessary to make a DinnerOptionResearchRequest.
    """
    type: Literal["slack-response"]
    response: str | list[dict[str, Any]]
```

**Fields:**
- `type`: Always "slack-response" (discriminator)
- `response`: Either markdown text or Slack Block Kit blocks

**Use Case:** Asking clarifying questions, providing simple responses without research.

#### `DinnerOptionResearchRequest`

Indicates the dispatcher should delegate to the research agent.

**Source:** `pydantic_temporal_example/agents/dispatch_agent.py` (lines 43-60)

```python
@dataclass
@with_config(use_attribute_docstrings=True)
class DinnerOptionResearchRequest:
    """A marker that indicates that you are ready to delegate to the dinner
    options research agent.
    """
    type: Literal["dinner-option-research-request"]
    location: WebSearchUserLocation
    cuisine_preferences: str
    price_preferences: str
    dietary_restrictions: str
    extra_info: str | None
```

**Fields:**
- `type`: Always "dinner-option-research-request"
- `location`: User's location for restaurant search
- `cuisine_preferences`: Preferred cuisines
- `price_preferences`: Budget preferences
- `dietary_restrictions`: Dietary requirements
- `extra_info`: Additional context (optional)

**Use Case:** When the dispatcher has collected enough information to delegate to research.

#### `DinnerOption`

Information about a single restaurant option.

**Source:** `pydantic_temporal_example/agents/dinner_research_agent.py` (lines 12-17)

```python
@dataclass
class DinnerOption:
    restaurant_name: str
    restaurant_address: str | None
    recommended_dishes: str
```

**Fields:**
- `restaurant_name`: Name of the restaurant
- `restaurant_address`: Address (optional)
- `recommended_dishes`: Suggested dishes at this restaurant

#### `DinnerSuggestions`

Final output from the research agent with formatted suggestions.

**Source:** `pydantic_temporal_example/agents/dinner_research_agent.py` (lines 19-33)

```python
@dataclass
@with_config(use_attribute_docstrings=True)
class DinnerSuggestions:
    response: str | list[dict[str, Any]]
    """The formatted message to show to the user.

    This should either be a markdown text string, or valid Slack blockkit blocks.

    The message should reference a list of suggestions where each suggestion includes:
    * restaurant name
    * restaurant address
    * any recommended dishes specific to that restaurant
    """
```

**Fields:**
- `response`: Formatted message with restaurant suggestions (text or Block Kit)

---

## API Endpoints

### POST /slack-events

Main webhook endpoint for receiving Slack Events API callbacks.

**Source:** `pydantic_temporal_example/api.py` (lines 23-48)

**Route:** `/slack-events`
**Method:** `POST`
**Content-Type:** `application/json`

**Function Signature:**

```python
async def handle_event(
    *,
    temporal_client: TemporalClient = Depends(get_temporal_client),
    slack_bot_user_id: str = Depends(get_slack_bot_user_id),
    body: SlackEventsAPIBody | URLVerificationEvent | dict[str, Any] = Depends(get_verified_slack_events_body),
) -> Response
```

**Parameters:**
- `temporal_client`: Temporal client for starting/signaling workflows (injected)
- `slack_bot_user_id`: Bot's user ID to ignore own messages (injected)
- `body`: Parsed and verified Slack event (injected)

**Returns:** `Response` with status code 204 (no content) or challenge response

**Behavior:**
1. Verifies Slack request signature (via dependency)
2. Handles URL verification challenges
3. Routes app mentions to workflow creation
4. Routes thread messages to existing workflows via signals
5. Ignores messages from the bot itself

**Request Headers Required:**
- `x-slack-request-timestamp`: Request timestamp
- `x-slack-signature`: HMAC signature for verification

**Example Slack Event:**

```json
{
  "token": "verification_token",
  "team_id": "T12345",
  "api_app_id": "A12345",
  "event": {
    "type": "app_mention",
    "user": "U12345",
    "text": "<@U67890> What should I eat?",
    "ts": "1234567890.123456",
    "channel": "C12345",
    "event_ts": "1234567890.123456"
  },
  "type": "event_callback",
  "event_id": "Ev12345",
  "event_time": 1234567890
}
```

**URL Verification Example:**

```json
{
  "type": "url_verification",
  "token": "abc123",
  "challenge": "xyz789"
}
```

Response:
```json
{
  "challenge": "xyz789"
}
```

**Error Responses:**
- `401 Unauthorized`: Invalid or missing signature, timestamp too old
- `204 No Content`: Event processed successfully

---

## Temporal Components

### Workflows

#### `SlackThreadWorkflow`

Long-running workflow that manages a single Slack thread conversation.

**Source:** `pydantic_temporal_example/temporal/workflows.py` (lines 35-108)

**Class Definition:**

```python
@workflow.defn
class SlackThreadWorkflow:
    def __init__(self) -> None:
        self._pending_events: asyncio.Queue[AppMentionEvent | MessageChannelsEvent] = asyncio.Queue()
        self._thread_messages: list[dict[str, Any]] = []
```

**Instance Variables:**
- `_pending_events`: Queue of incoming Slack events to process
- `_thread_messages`: Accumulated thread message history

**Methods:**

##### `run()`

Main workflow execution loop.

**Source:** Lines 48-55

```python
@workflow.run
async def run(self) -> None
```

**Behavior:**
- Runs indefinitely, processing events from the queue
- Waits for new events using `workflow.wait_condition`
- Processes each event via `handle_event()`

##### `submit_app_mention_event()`

Signal handler for app mention events (starts workflow).

**Source:** Lines 56-58

```python
@workflow.signal
async def submit_app_mention_event(self, event: AppMentionEvent)
```

**Parameters:**
- `event`: App mention event from Slack

**Behavior:**
- Adds event to processing queue
- This is used as the workflow's start signal

##### `submit_message_channels_event()`

Signal handler for subsequent messages in the thread.

**Source:** Lines 60-62

```python
@workflow.signal
async def submit_message_channels_event(self, event: MessageChannelsEvent)
```

**Parameters:**
- `event`: Message event from Slack

**Behavior:**
- Adds event to processing queue
- Called when users reply to a thread where bot was mentioned

##### `handle_event()`

Processes a single event from the queue.

**Source:** Lines 64-107

```python
async def handle_event(self, event: AppMentionEvent | MessageChannelsEvent)
```

**Parameters:**
- `event`: Event to process

**Behavior:**
1. Adds "spin" reaction to show bot is thinking
2. Fetches new messages from the thread
3. Calls `handle_user_request()` to run agents
4. Posts response to Slack (if applicable)
5. Removes "spin" reaction

**Example Usage:**

```python
from temporalio.client import Client
from pydantic_temporal_example.temporal.workflows import SlackThreadWorkflow
from pydantic_temporal_example.models import AppMentionEvent

client = await Client.connect("localhost:7233")

# Start workflow for a new thread
event = AppMentionEvent(
    type="app_mention",
    user="U12345",
    text="@bot help with dinner",
    ts="1234567890.123456",
    channel="C12345",
    event_ts="1234567890.123456"
)

workflow_id = f"app-mention-{event.reply_thread_ts.replace('.', '-')}"
await client.start_workflow(
    SlackThreadWorkflow.run,
    id=workflow_id,
    start_signal="submit_app_mention_event",
    start_signal_args=[event],
    task_queue="agent-task-queue"
)

# Signal existing workflow with new message
handle = client.get_workflow_handle_for(
    SlackThreadWorkflow.run,
    workflow_id=workflow_id
)
await handle.signal("submit_message_channels_event", args=[message_event])
```

#### `handle_user_request()`

Orchestrates the dispatcher and research agents.

**Source:** `pydantic_temporal_example/temporal/workflows.py` (lines 110-118)

```python
@logfire.instrument
async def handle_user_request(stringified_thread: str) -> NoResponse | SlackResponse | DinnerSuggestions
```

**Parameters:**
- `stringified_thread`: JSON string of thread messages

**Returns:** One of:
- `NoResponse`: No action needed
- `SlackResponse`: Direct response from dispatcher
- `DinnerSuggestions`: Research results

**Behavior:**
1. Runs dispatcher agent to determine intent
2. If research needed, runs dinner research agent
3. Returns appropriate response type

**Example:**

```python
import json
from pydantic_temporal_example.temporal.workflows import handle_user_request

thread = [
    {"user": "U12345", "text": "What should I eat?"},
    {"user": "U67890", "text": "What's your budget?"},
    {"user": "U12345", "text": "Under $30, I like Italian"}
]

result = await handle_user_request(json.dumps(thread))
if isinstance(result, DinnerSuggestions):
    print("Got research results!")
```

### Activities

All Slack API interactions are implemented as Temporal activities for durability and retryability.

**Source:** `pydantic_temporal_example/temporal/slack_activities.py`

#### `slack_conversations_replies()`

Fetches messages from a Slack thread.

**Source:** Lines 12-28

```python
@activity.defn
@logfire.instrument
async def slack_conversations_replies(
    request: SlackConversationsRepliesRequest
) -> list[dict[str, Any]]
```

**Parameters:**
- `request`: Request with channel, thread timestamp, and optional oldest timestamp

**Returns:** List of message dictionaries from Slack API

**Behavior:**
- Paginates through thread messages
- Respects `oldest` parameter to fetch only new messages
- Handles Slack's cursor-based pagination

**Example:**

```python
from pydantic_temporal_example.temporal.slack_activities import slack_conversations_replies
from pydantic_temporal_example.models import SlackConversationsRepliesRequest

request = SlackConversationsRepliesRequest(
    channel="C12345",
    ts="1234567890.123456",
    oldest="1234567800.000000"
)

messages = await slack_conversations_replies(request)
# [{"user": "U12345", "text": "Hello", "ts": "1234567890.123456"}, ...]
```

#### `slack_chat_post_message()`

Posts a message to a Slack thread.

**Source:** Lines 31-40

```python
@activity.defn
@logfire.instrument
async def slack_chat_post_message(reply: SlackReply) -> dict[str, Any]
```

**Parameters:**
- `reply`: Reply containing thread info and content

**Returns:** Slack API response data

**Behavior:**
- Posts to specified thread
- Supports both text and Block Kit formats
- Returns message metadata from Slack

**Example:**

```python
from pydantic_temporal_example.temporal.slack_activities import slack_chat_post_message
from pydantic_temporal_example.models import SlackReply, SlackMessageID

reply = SlackReply(
    thread=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    content="Here are my suggestions!"
)

result = await slack_chat_post_message(reply)
# {"ok": True, "ts": "1234567891.123456", ...}
```

#### `slack_chat_delete()`

Deletes a Slack message.

**Source:** Lines 43-50

```python
@activity.defn
@logfire.instrument
async def slack_chat_delete(message: SlackMessageID) -> dict[str, Any]
```

**Parameters:**
- `message`: Message identifier (channel and timestamp)

**Returns:** Slack API response data

**Example:**

```python
from pydantic_temporal_example.temporal.slack_activities import slack_chat_delete
from pydantic_temporal_example.models import SlackMessageID

message = SlackMessageID(channel="C12345", ts="1234567890.123456")
result = await slack_chat_delete(message)
```

#### `slack_reactions_add()`

Adds a reaction emoji to a message.

**Source:** Lines 53-61

```python
@activity.defn
@logfire.instrument
async def slack_reactions_add(reaction: SlackReaction) -> dict[str, Any]
```

**Parameters:**
- `reaction`: Reaction with message ID and emoji name

**Returns:** Slack API response data

**Example:**

```python
from pydantic_temporal_example.temporal.slack_activities import slack_reactions_add
from pydantic_temporal_example.models import SlackReaction, SlackMessageID

reaction = SlackReaction(
    message=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    name="spin"  # Shows :spin: emoji
)

await slack_reactions_add(reaction)
```

#### `slack_reactions_remove()`

Removes a reaction emoji from a message.

**Source:** Lines 64-72

```python
@activity.defn
@logfire.instrument
async def slack_reactions_remove(reaction: SlackReaction) -> dict[str, Any]
```

**Parameters:**
- `reaction`: Reaction with message ID and emoji name

**Returns:** Slack API response data

**Example:**

```python
from pydantic_temporal_example.temporal.slack_activities import slack_reactions_remove
from pydantic_temporal_example.models import SlackReaction, SlackMessageID

reaction = SlackReaction(
    message=SlackMessageID(channel="C12345", ts="1234567890.123456"),
    name="spin"
)

await slack_reactions_remove(reaction)
```

### Temporal Client

#### `build_temporal_client()`

Creates and configures a Temporal client with PydanticAI and Logfire plugins.

**Source:** `pydantic_temporal_example/temporal/client.py` (lines 8-21)

```python
async def build_temporal_client() -> TemporalClient
```

**Returns:** `TemporalClient` - Configured Temporal client instance

**Behavior:**
- Connects to Temporal server (local or remote based on settings)
- Installs PydanticAI plugin for agent execution
- Installs Logfire plugin for observability
- Configures Logfire instrumentation

**Example:**

```python
from pydantic_temporal_example.temporal.client import build_temporal_client

client = await build_temporal_client()
# Use client to start workflows, send signals, etc.
```

### Temporal Worker

#### `temporal_worker()`

Context manager that creates and manages a Temporal worker.

**Source:** `pydantic_temporal_example/temporal/worker.py` (lines 17-36)

```python
@asynccontextmanager
async def temporal_worker() -> AsyncIterator[Worker]
```

**Returns:** `AsyncIterator[Worker]` - Temporal worker instance

**Behavior:**
- Starts local Temporal server if `temporal_host` is None
- Creates worker with workflows and activities registered
- Registers agent plugins for dispatcher and research agents
- Cleans up resources on exit

**Example:**

```python
from pydantic_temporal_example.temporal.worker import temporal_worker
import uvicorn

async def main():
    async with temporal_worker() as worker:
        # Worker is running, now start your app
        config = uvicorn.Config("app:app", port=4000)
        server = uvicorn.Server(config)
        await server.serve()
```

---

## AI Agents

### Dispatcher Agent

Fast, lightweight agent that routes requests and determines intent.

**Source:** `pydantic_temporal_example/agents/dispatch_agent.py` (lines 64-94)

**Agent Configuration:**

```python
dispatch_agent = Agent(
    model="openai-responses:gpt-5-mini",
    output_type=[NoResponse, SlackResponse, DinnerOptionResearchRequest],
    instructions="""...""",
    tools=[duckduckgo_search_tool()]
)
```

**Configuration:**
- **Model:** `openai-responses:gpt-5-mini` (fast, cost-effective)
- **Output Types:** Structured union of NoResponse, SlackResponse, or DinnerOptionResearchRequest
- **Tools:** DuckDuckGo search for quick lookups

**Instructions Summary:**
- Routes Slack thread conversations
- Collects information for dinner research
- Asks clarifying questions when information is missing
- Delegates to research agent when ready

**Temporal Wrapper:**

```python
from pydantic_temporal_example.temporal.workflows import temporal_dispatch_agent

# Use in workflows
result = await temporal_dispatch_agent.run("Thread messages JSON")
```

**Example Usage:**

```python
from pydantic_temporal_example.agents.dispatch_agent import dispatch_agent
import json

thread = json.dumps([
    {"user": "U12345", "text": "@bot I want Italian food in Boston"}
])

result = await dispatch_agent.run(thread)

if isinstance(result.output, DinnerOptionResearchRequest):
    print(f"Location: {result.output.location}")
    print(f"Cuisine: {result.output.cuisine_preferences}")
```

**Output Types:**

1. **NoResponse:** Users are talking among themselves
2. **SlackResponse:** Quick reply or clarifying question
3. **DinnerOptionResearchRequest:** Ready for research with all info collected

### Dinner Research Agent

Thorough research agent that finds and suggests restaurants.

**Source:** `pydantic_temporal_example/agents/dinner_research_agent.py` (lines 35-43)

**Agent Configuration:**

```python
dinner_research_agent = Agent(
    model="openai-responses:gpt-5-mini",
    output_type=NativeOutput(DinnerSuggestions),
    instructions="""...""",
    tools=[duckduckgo_search_tool()]
)
```

**Configuration:**
- **Model:** `openai-responses:gpt-5-mini` (more capable for research)
- **Output Type:** DinnerSuggestions with formatted response
- **Tools:** DuckDuckGo search for restaurant research

**Instructions Summary:**
- Research local dinner options based on user preferences
- Use web search to find current, accurate information
- Format results with restaurant names, addresses, and dish recommendations

**Temporal Wrapper:**

```python
from pydantic_temporal_example.temporal.workflows import temporal_dinner_research_agent

# Use in workflows
result = await temporal_dinner_research_agent.run(user_preferences)
```

**Example Usage:**

```python
from pydantic_temporal_example.agents.dinner_research_agent import dinner_research_agent
from pydantic_ai import WebSearchUserLocation

preferences = {
    "location": WebSearchUserLocation(city="Boston", state="MA"),
    "cuisine_preferences": "Italian",
    "dietary_restrictions": "vegetarian",
    "price_preferences": "under $30"
}

result = await dinner_research_agent.run(str(preferences))

if isinstance(result.output, DinnerSuggestions):
    print(result.output.response)  # Formatted restaurant suggestions
```

---

## Utility Functions

### Slack Utilities

#### `get_verified_slack_events_body()`

FastAPI dependency that verifies and parses Slack webhook requests.

**Source:** `pydantic_temporal_example/slack.py` (lines 13-48)

```python
async def get_verified_slack_events_body(
    request: Request,
) -> SlackEventsAPIBody | URLVerificationEvent | dict[str, Any]
```

**Parameters:**
- `request`: FastAPI/Starlette request object

**Returns:** Parsed and validated Slack event

**Raises:**
- `HTTPException(401)`: If signature verification fails or timestamp is too old

**Security Checks:**
1. Validates `x-slack-request-timestamp` header exists
2. Checks timestamp is not older than 5 minutes (replay attack prevention)
3. Validates `x-slack-signature` header exists
4. Computes HMAC-SHA256 signature
5. Compares signatures using constant-time comparison

**Example Usage:**

```python
from fastapi import FastAPI, Depends
from pydantic_temporal_example.slack import get_verified_slack_events_body

app = FastAPI()

@app.post("/slack-events")
async def handle_event(
    body = Depends(get_verified_slack_events_body)
):
    # body is verified and parsed
    return {"ok": True}
```

**Signature Verification Algorithm:**

```python
base_string = f"v0:{timestamp}:{request_body}"
expected_sig = "v0=" + hmac.new(
    signing_secret.encode("utf-8"),
    base_string.encode("utf-8"),
    hashlib.sha256
).hexdigest()
```

### Dependencies

#### `lifespan()`

FastAPI lifespan context manager for application startup/shutdown.

**Source:** `pydantic_temporal_example/dependencies.py` (lines 14-20)

```python
@asynccontextmanager
async def lifespan(_server: FastAPI) -> AsyncIterator[dict[str, Any]]
```

**Parameters:**
- `_server`: FastAPI application instance

**Yields:** Dictionary with:
- `temporal_client`: Temporal client instance
- `slack_bot_user_id`: Bot's user ID

**Behavior:**
- Creates Slack client
- Fetches bot's user ID via `auth_test` API
- Creates Temporal client
- Provides resources to FastAPI application state

**Example:**

```python
from fastapi import FastAPI
from pydantic_temporal_example.dependencies import lifespan

app = FastAPI(lifespan=lifespan)

# Resources available in request.state.temporal_client
# and request.state.slack_bot_user_id
```

#### `get_temporal_client()`

FastAPI dependency that retrieves the Temporal client from request state.

**Source:** `pydantic_temporal_example/dependencies.py` (lines 23-24)

```python
async def get_temporal_client(request: Request) -> TemporalClient
```

**Parameters:**
- `request`: FastAPI request

**Returns:** `TemporalClient` instance

**Example:**

```python
from fastapi import Depends
from pydantic_temporal_example.dependencies import get_temporal_client

@app.post("/endpoint")
async def my_endpoint(
    client = Depends(get_temporal_client)
):
    # Use client to interact with Temporal
    pass
```

#### `get_slack_bot_user_id()`

FastAPI dependency that retrieves the bot's user ID from request state.

**Source:** `pydantic_temporal_example/dependencies.py` (lines 27-28)

```python
async def get_slack_bot_user_id(request: Request) -> str
```

**Parameters:**
- `request`: FastAPI request

**Returns:** Slack bot user ID

**Example:**

```python
from fastapi import Depends
from pydantic_temporal_example.dependencies import get_slack_bot_user_id

@app.post("/endpoint")
async def my_endpoint(
    bot_id = Depends(get_slack_bot_user_id)
):
    # Use bot_id to filter messages
    if message.user == bot_id:
        # Ignore own messages
        pass
```

---

## Usage Examples

### Basic Usage

#### Running the Application

```python
import asyncio
from pydantic_temporal_example.app import main

# Start the application with Temporal worker and FastAPI server
asyncio.run(main())
```

This:
1. Starts Temporal worker (or local dev server if configured)
2. Starts FastAPI server on port 4000
3. Begins listening for Slack events

#### Environment Setup

```bash
# .env file
SLACK_BOT_TOKEN=xoxb-your-token-here
SLACK_SIGNING_SECRET=your-signing-secret
OPENAI_API_KEY=sk-your-openai-key
TEMPORAL_HOST=  # Leave empty for local dev server
TEMPORAL_PORT=7233
TEMPORAL_TASK_QUEUE=agent-task-queue
```

#### Handling an App Mention

```python
from pydantic_temporal_example.models import AppMentionEvent
from pydantic_temporal_example.temporal.workflows import SlackThreadWorkflow
from temporalio.client import Client

# Event received from Slack
event = AppMentionEvent(
    type="app_mention",
    user="U12345",
    text="@dinnerbot I want Thai food in Seattle",
    ts="1234567890.123456",
    channel="C12345",
    event_ts="1234567890.123456"
)

# Start workflow
client = await Client.connect("localhost:7233")
workflow_id = f"app-mention-{event.reply_thread_ts.replace('.', '-')}"

await client.start_workflow(
    SlackThreadWorkflow.run,
    id=workflow_id,
    start_signal="submit_app_mention_event",
    start_signal_args=[event],
    task_queue="agent-task-queue"
)
```

### Advanced Patterns

#### Multi-Turn Conversation

```python
# User mentions bot
app_mention = AppMentionEvent(
    type="app_mention",
    user="U12345",
    text="@bot help with dinner",
    ts="1000.000",
    channel="C12345",
    event_ts="1000.000"
)

# Start workflow
await client.start_workflow(
    SlackThreadWorkflow.run,
    id="app-mention-1000-000",
    start_signal="submit_app_mention_event",
    start_signal_args=[app_mention],
    task_queue="agent-task-queue"
)

# Bot asks: "What city are you in?"
# User replies in thread
reply1 = MessageChannelsEvent(
    type="message",
    user="U12345",
    text="Seattle",
    ts="1001.000",
    channel="C12345",
    event_ts="1001.000",
    thread_ts="1000.000"  # References original message
)

# Signal existing workflow
handle = client.get_workflow_handle_for(
    SlackThreadWorkflow.run,
    workflow_id="app-mention-1000-000"
)
await handle.signal("submit_message_channels_event", args=[reply1])

# Bot asks: "What cuisine?"
# User replies
reply2 = MessageChannelsEvent(
    type="message",
    user="U12345",
    text="Thai, under $25",
    ts="1002.000",
    channel="C12345",
    event_ts="1002.000",
    thread_ts="1000.000"
)

await handle.signal("submit_message_channels_event", args=[reply2])
# Bot now has enough info and triggers research agent
```

#### Custom Agent Integration

```python
from pydantic_ai import Agent
from pydantic_ai.durable_exec.temporal import TemporalAgent
from temporalio.worker import Worker

# Define custom agent
custom_agent = Agent(
    model="openai-responses:gpt-5",
    output_type=CustomOutput,
    instructions="Custom instructions..."
)

# Wrap for Temporal
temporal_custom_agent = TemporalAgent(custom_agent, name="custom_agent")

# Add to worker
worker = Worker(
    client,
    task_queue="agent-task-queue",
    workflows=[SlackThreadWorkflow],
    activities=ALL_SLACK_ACTIVITIES,
    plugins=[
        AgentPlugin(temporal_dispatch_agent),
        AgentPlugin(temporal_dinner_research_agent),
        AgentPlugin(temporal_custom_agent)  # Add custom agent
    ]
)
```

#### Manual Temporal Client Usage

```python
from pydantic_temporal_example.temporal.client import build_temporal_client
from pydantic_temporal_example.temporal.workflows import SlackThreadWorkflow

# Build client manually
client = await build_temporal_client()

# Query workflow state
handle = client.get_workflow_handle_for(
    SlackThreadWorkflow.run,
    workflow_id="app-mention-1000-000"
)

# Check if workflow is running
try:
    info = await handle.describe()
    print(f"Workflow status: {info.status}")
except Exception:
    print("Workflow not found")

# Send custom signal
await handle.signal("submit_app_mention_event", args=[event])

# Terminate workflow
await handle.terminate("User requested cancellation")
```

#### Block Kit Response

```python
from pydantic_temporal_example.models import SlackReply, SlackMessageID

# Create rich Block Kit response
blocks = [
    {
        "type": "header",
        "text": {
            "type": "plain_text",
            "text": "Dinner Suggestions"
        }
    },
    {
        "type": "section",
        "text": {
            "type": "mrkdwn",
            "text": "*Restaurant A*\n123 Main St\nRecommended: Pad Thai, Tom Yum"
        }
    },
    {
        "type": "divider"
    },
    {
        "type": "section",
        "text": {
            "type": "mrkdwn",
            "text": "*Restaurant B*\n456 Oak Ave\nRecommended: Green Curry, Spring Rolls"
        }
    }
]

reply = SlackReply(
    thread=SlackMessageID(channel="C12345", ts="1000.000"),
    content=blocks
)

# Post via activity
await workflow.execute_activity(
    slack_chat_post_message,
    reply,
    start_to_close_timeout=timedelta(seconds=10)
)
```

---

## Best Practices

### Agent Design

1. **Use Structured Outputs**: Always define explicit Pydantic models for agent outputs instead of relying on free-text responses.

```python
# Good
output_type=DinnerSuggestions

# Bad
output_type=str  # Then parse the string yourself
```

2. **Multi-Agent Architecture**: Use lightweight agents for routing/dispatch and more capable agents for complex tasks.

```python
# Fast dispatcher
dispatch_agent = Agent(model="gpt-5-mini", ...)

# Thorough researcher
research_agent = Agent(model="gpt-5", ...)
```

3. **Clear Instructions**: Provide detailed, specific instructions in agent definitions.

```python
instructions="""
You are a dispatch agent for dinner recommendations.

Rules:
- If users are talking among themselves, return NoResponse
- If you need more info, return SlackResponse with questions
- Never guess preferences - always ask
"""
```

### Temporal Workflows

1. **One Workflow Per Entity**: Create one workflow instance per logical entity (e.g., one per Slack thread).

2. **Deterministic Workflow Code**: Never use random(), datetime.now(), or other non-deterministic functions directly in workflow code.

```python
# Bad - non-deterministic
async def run(self):
    timestamp = datetime.now()  # Don't do this!

# Good - use workflow utilities
async def run(self):
    timestamp = workflow.now()
```

3. **Use Activities for External Calls**: All Slack API calls, database queries, and HTTP requests should be activities.

```python
# Activities are retriable and durable
@activity.defn
async def slack_chat_post_message(reply: SlackReply):
    return await slack_client.chat_postMessage(...)
```

4. **Signal for Async Updates**: Use signals to push data to running workflows instead of polling.

```python
@workflow.signal
async def submit_message(self, event: MessageEvent):
    await self._queue.put(event)
```

### Error Handling

1. **Retry Activities**: Configure retry policies for activities that may fail transiently.

```python
await workflow.execute_activity(
    slack_chat_post_message,
    reply,
    start_to_close_timeout=timedelta(seconds=10),
    retry_policy=RetryPolicy(
        maximum_attempts=3,
        initial_interval=timedelta(seconds=1)
    )
)
```

2. **Graceful Degradation**: Handle agent failures gracefully with fallback responses.

```python
try:
    result = await temporal_dispatcher.run(thread)
except Exception as e:
    logfire.error("Agent failed", error=e)
    # Post error message to Slack
    await post_error_response()
```

3. **Validate Inputs**: Use Pydantic models to validate all inputs early.

```python
# Pydantic validates automatically
event = MessageChannelsEvent(**slack_data)  # Raises if invalid
```

### Security

1. **Always Verify Slack Signatures**: Never skip signature verification in production.

```python
# This is critical for security
body = Depends(get_verified_slack_events_body)
```

2. **Environment Variables**: Never hardcode secrets; always use environment variables.

```python
# Good
slack_bot_token: str  # Loaded from env

# Bad
slack_bot_token = "xoxb-hardcoded-token"  # Never do this!
```

3. **Timeout Protection**: Set reasonable timeouts on all activities.

```python
start_to_close_timeout=timedelta(seconds=10)  # Prevent hanging
```

### Performance

1. **Cache Settings**: Use `@cache` decorator for settings to avoid repeated loads.

```python
@cache
def get_settings() -> Settings:
    return Settings()
```

2. **Batch Operations**: Fetch thread messages with pagination instead of one-by-one.

```python
# Handles pagination internally
messages = await slack_conversations_replies(request)
```

3. **Async/Await**: Use async operations throughout for better concurrency.

```python
async def handle_event(self, event):
    # Run activities concurrently
    await asyncio.gather(
        workflow.execute_activity(add_reaction, ...),
        workflow.execute_activity(fetch_messages, ...)
    )
```

---

## Error Handling

### Common Error Types

#### Slack API Errors

**Error:** `SlackApiError`
**Cause:** Slack API call failed (rate limit, invalid token, etc.)
**Handled By:** Activities are automatically retried by Temporal
**Example:**

```python
from slack_sdk.errors import SlackApiError

try:
    await slack_client.chat_postMessage(...)
except SlackApiError as e:
    logfire.error("Slack API error", error=e)
    # Temporal will retry the activity
    raise
```

#### Signature Verification Errors

**Error:** `HTTPException(401)`
**Cause:** Invalid signature, missing headers, or timestamp too old
**Handled By:** `get_verified_slack_events_body()` dependency
**Prevention:**

```python
# Ensure headers are set
headers = {
    "x-slack-request-timestamp": "1234567890",
    "x-slack-signature": "v0=abc123..."
}

# Timestamp must be within 5 minutes
five_minutes_ago = int(time.time()) - (60 * 5)
if int(timestamp_header) < five_minutes_ago:
    raise HTTPException(401, detail="Request timestamp too old")
```

#### Workflow Errors

**Error:** `TemporalError`
**Cause:** Workflow doesn't exist, already completed, or failed
**Handled By:** Try/except blocks in API handlers
**Example:**

```python
from temporalio.exceptions import TemporalError

try:
    await handle.describe()
    await handle.signal("submit_message", args=[event])
except TemporalError:
    logfire.info("No workflow found for this thread")
    # This is expected for non-bot threads
    pass
```

#### Agent Execution Errors

**Error:** Various (API rate limits, model errors, parsing failures)
**Cause:** AI agent execution failures
**Handled By:** Logfire instrumentation + graceful fallbacks
**Example:**

```python
try:
    result = await temporal_dispatcher.run(thread)
except Exception as e:
    logfire.error("Agent execution failed", error=e, thread=thread)
    # Return error response to user
    return SlackResponse(
        type="slack-response",
        response="I'm having trouble processing your request. Please try again."
    )
```

### Error Recovery Patterns

#### Activity Retries

Temporal automatically retries failed activities:

```python
await workflow.execute_activity(
    slack_chat_post_message,
    reply,
    start_to_close_timeout=timedelta(seconds=10),
    # Default retry policy retries transient failures
)
```

#### Workflow Recovery

Workflows automatically resume after server restarts:

```python
# Workflow state is persisted
# If server crashes during execution, workflow continues
# from last successful checkpoint
@workflow.run
async def run(self):
    # This runs to completion even across restarts
    while True:
        await self.handle_event(...)
```

#### Graceful Degradation

```python
async def handle_event(self, event):
    try:
        # Try to process normally
        result = await handle_user_request(thread)
    except Exception as e:
        logfire.error("Request handling failed", error=e)
        # Post generic error message
        result = SlackResponse(
            type="slack-response",
            response="Sorry, I encountered an error. Please try again."
        )

    # Always try to respond
    try:
        await post_response(result)
    except Exception as e:
        logfire.error("Failed to post response", error=e)
        # Last resort - add reaction to show error
        await add_reaction(event, "x")
```

### Monitoring and Debugging

#### Logfire Integration

The application uses Logfire for comprehensive observability:

```python
import logfire

# Configure on startup
logfire.configure(service_name="app")
logfire.instrument_pydantic_ai()
logfire.instrument_httpx(capture_all=True)
logfire.instrument_fastapi(app)

# Instrument functions
@logfire.instrument
async def handle_user_request(thread: str):
    # Automatically logged with inputs/outputs
    pass
```

**What Gets Logged:**
- All agent executions with prompts and responses
- All HTTP requests (Slack API, OpenAI API)
- All FastAPI requests and responses
- Custom log messages

**Example Log Query:**

```python
# View all agent executions
logfire.search('span.name == "Agent.run"')

# View failed Slack API calls
logfire.search('service.name == "app" AND error == true AND span.name contains "slack"')
```

#### Temporal Web UI

Access workflow state via Temporal Web UI:

```bash
# Start local dev server with UI
temporal server start-dev --ui-port 8233

# Navigate to http://localhost:8233
```

**What You Can See:**
- Running workflows
- Workflow history and events
- Activity executions and retries
- Signal history
- Workflow state

---

## Conclusion

This API reference provides complete documentation for building and extending the PydanticAI Temporal Example application. For additional help:

- **Source Code:** All file paths are referenced with line numbers
- **README:** `README.md`
- **PydanticAI Docs:** https://ai.pydantic.dev/
- **Temporal Docs:** https://docs.temporal.io/
- **Slack API Docs:** https://api.slack.com/

Key architectural principles:
1. Type-safe AI outputs with Pydantic models
2. Durable execution with Temporal workflows
3. One workflow per conversation thread
4. Multi-agent design (dispatcher + specialist)
5. Activities for all external I/O
6. Comprehensive observability with Logfire
