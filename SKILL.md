---
name: view-chat
description: Log and retrieve chatbot conversations via the View Chat API. Use when an AI agent needs to store conversation messages, organize them into groups, retrieve conversation history, or query conversations by time range. Triggers on requests like "log this conversation", "save chat messages", "get conversation history", "list conversations", or any task requiring conversation monitoring and storage.
---

# View Chat -- Conversation Monitoring API

View Chat lets AI agents log chatbot conversations in real-time and retrieve
them later. Conversations are organized into groups for multi-tenant or
multi-project setups. A web UI at `https://conversation-viewer.onrender.com`
provides a dashboard for browsing logged conversations.

Base URL: `https://api.view-chat.com`

## Authentication

This skill requires the `VIEW_CHAT_TOKEN` environment variable to be set. Read
it with `process.env.VIEW_CHAT_TOKEN` (Node/Deno) or `$VIEW_CHAT_TOKEN` (shell).
If the variable is not set, stop and ask the user to provide it.

## Protocol

All requests are `POST` to the base URL with a JSON body:

```json
{
  "endpoint": "<endpoint_name>",
  "token": "<your_api_token>",
  "payload": { ... }
}
```

On success, responses include `{ "ok": true, ... }`. On error, the API returns
HTTP 400 with `{ "error": "message" }`.

## Quick start

### Log a message

```bash
curl -X POST https://api.view-chat.com \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "log",
    "token": "YOUR_API_TOKEN",
    "payload": {
      "event": {
        "from": "user",
        "text": "Hello, how can I help?",
        "time": 1710000000000
      },
      "conversation": "conv-123",
      "groups": ["support-bot"]
    }
  }'
```

Response: `{"ok": true}`

### Get conversations

```bash
curl -X POST https://api.view-chat.com \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "getConversations",
    "token": "YOUR_API_TOKEN",
    "payload": {}
  }'
```

### Get messages for a conversation

```bash
curl -X POST https://api.view-chat.com \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "getMessages",
    "token": "YOUR_API_TOKEN",
    "payload": {
      "conversationId": "CONVERSATION_ID"
    }
  }'
```

## Endpoints

### log

Log a message to a conversation. The conversation and groups are created
automatically if they don't exist.

| Field             | Type     | Required | Description                                                                        |
| ----------------- | -------- | -------- | ---------------------------------------------------------------------------------- |
| `event.from`      | string   | yes      | Who sent the message                                                               |
| `event.text`      | string   | yes      | Message content                                                                    |
| `event.time`      | number   | yes      | Unix timestamp in milliseconds                                                     |
| `event.foreignId` | string   | no       | Client-provided unique message ID for deduplication. If a message with this ID already exists in the conversation, the log is silently ignored. |
| `conversation`    | string   | yes      | Conversation identifier (created automatically if new)                             |
| `groups`          | string[] | no       | Group names to assign (created automatically). Mutually exclusive with `groupIds`. |
| `groupIds`        | string[] | no       | Group IDs to assign (must already exist). Mutually exclusive with `groups`.        |

Response: `{"ok": true}`

> **Idempotent logging**: If you pass `event.foreignId`, View Chat deduplicates
> by that ID within the conversation. Subsequent logs with the same
> `conversation` + `foreignId` are silently ignored. This lets you retry failed
> log calls without creating duplicate messages.

```typescript
await fetch("https://api.view-chat.com", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    endpoint: "log",
    token: API_TOKEN,
    payload: {
      event: { from: "user", text: "What is my balance?", time: Date.now() },
      conversation: "session-abc",
      groups: ["finance-bot"],
    },
  }),
});
```

### getConversations

List conversations for the authenticated account, with optional filtering and
pagination.

| Field     | Type   | Required | Description                                                  |
| --------- | ------ | -------- | ------------------------------------------------------------ |
| `groupId` | string | no       | Filter by group ID                                           |
| `limit`   | number | no       | Max conversations to return (default: 25, max: 100)          |
| `offset`  | number | no       | Number of conversations to skip (default: 0)                 |
| `since`   | number | no       | Only conversations with `lastMessageTime >= since` (Unix ms) |
| `until`   | number | no       | Only conversations with `lastMessageTime <= until` (Unix ms) |

Response:

```json
{
  "ok": true,
  "conversations": [
    {
      "id": "conv-id",
      "name": "session-abc",
      "lastMessageTime": 1710000000000,
      "groups": [{ "id": "group-id", "name": "finance-bot" }]
    }
  ]
}
```

### getMessages

Get messages for a specific conversation, with pagination.

| Field            | Type   | Required | Description                                     |
| ---------------- | ------ | -------- | ----------------------------------------------- |
| `conversationId` | string | yes      | Conversation ID                                 |
| `limit`          | number | no       | Max messages to return (default: 100, max: 200) |
| `offset`         | number | no       | Number of messages to skip (default: 0)         |

Response:

```json
{
  "ok": true,
  "messages": [
    {
      "id": "msg-id",
      "from": "user",
      "text": "What is my balance?",
      "time": 1710000000000
    }
  ]
}
```

Messages are ordered by time descending (newest first).

### createGroup

Create a named group for organizing conversations.

| Field  | Type   | Required | Description            |
| ------ | ------ | -------- | ---------------------- |
| `name` | string | yes      | Name for the new group |

Response: `{"ok": true, "groupId": "..."}`

### addViewer

Grant a user read access to a group's conversations via the web UI.

| Field     | Type   | Required | Description                        |
| --------- | ------ | -------- | ---------------------------------- |
| `email`   | string | yes      | Email address of the viewer        |
| `groupId` | string | yes      | ID of the group to grant access to |

Response: `{"ok": true}`

## Common patterns

### Log a full conversation turn-by-turn

```typescript
const TOKEN = process.env.VIEW_CHAT_TOKEN;
const CONVERSATION = `session-${Date.now()}`;

const logMessage = (from: string, text: string) =>
  fetch("https://api.view-chat.com", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      endpoint: "log",
      token: TOKEN,
      payload: {
        event: { from, text, time: Date.now() },
        conversation: CONVERSATION,
        groups: ["my-bot"],
      },
    }),
  });

await logMessage("user", "How do I reset my password?");
await logMessage("assistant", "Go to Settings > Security > Reset Password.");
await logMessage("user", "Thanks!");
```

### Paginate through all conversations

```typescript
const allConversations = [];
let offset = 0;
const limit = 100;

while (true) {
  const res = await fetch("https://api.view-chat.com", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      endpoint: "getConversations",
      token: TOKEN,
      payload: { limit, offset },
    }),
  }).then((r) => r.json());

  allConversations.push(...res.conversations);
  if (res.conversations.length < limit) break;
  offset += limit;
}
```

### Get conversations from the last 24 hours

```typescript
const since = Date.now() - 24 * 60 * 60 * 1000;

const res = await fetch("https://api.view-chat.com", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    endpoint: "getConversations",
    token: TOKEN,
    payload: { since },
  }),
}).then((r) => r.json());
```

### Fetch full conversation history

```typescript
const allMessages = [];
let offset = 0;
const limit = 200;

while (true) {
  const res = await fetch("https://api.view-chat.com", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      endpoint: "getMessages",
      token: TOKEN,
      payload: { conversationId: "CONV_ID", limit, offset },
    }),
  }).then((r) => r.json());

  allMessages.push(...res.messages);
  if (res.messages.length < limit) break;
  offset += limit;
}

// Messages are newest-first, reverse for chronological order
allMessages.reverse();
```

### Set up a new project with a group and viewer

```typescript
// 1. Create a group
const groupRes = await fetch("https://api.view-chat.com", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    endpoint: "createGroup",
    token: TOKEN,
    payload: { name: "customer-support" },
  }),
}).then((r) => r.json());

const groupId = groupRes.groupId;

// 2. Add a viewer
await fetch("https://api.view-chat.com", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    endpoint: "addViewer",
    token: TOKEN,
    payload: { email: "teammate@example.com", groupId },
  }),
});

// 3. Log messages using the group ID
await fetch("https://api.view-chat.com", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    endpoint: "log",
    token: TOKEN,
    payload: {
      event: { from: "bot", text: "Welcome!", time: Date.now() },
      conversation: "first-conversation",
      groupIds: [groupId],
    },
  }),
});
```

## Key concepts

- **Conversation**: A named thread of messages. Created automatically on first
  `log` call with that name. The `name` you pass as `conversation` becomes the
  conversation's display name.
- **Group**: An organizational container for conversations. Use groups to
  separate conversations by project, bot, environment, etc. A conversation can
  belong to multiple groups.
- **Viewer**: A user (by email) granted read access to a group's conversations
  through the web dashboard.
- **Token**: API authentication credential. Tokens are managed through the web
  UI at `https://conversation-viewer.onrender.com`, not through the API.

## Error handling

The API returns HTTP 400 with `{"error": "..."}` for all errors. Common errors:

| Error                   | Cause                                           |
| ----------------------- | ----------------------------------------------- |
| Invalid token           | Token is malformed, revoked, or doesn't exist   |
| Missing required fields | A required payload field is missing             |
| Invalid input format    | Payload doesn't match the expected schema       |
| Conversation not found  | `conversationId` in `getMessages` doesn't exist |
