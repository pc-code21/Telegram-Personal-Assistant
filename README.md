# Telegram Personal Assistant

An AI-powered personal assistant built in [n8n](https://n8n.io) that lives inside Telegram. It understands both text and voice messages, remembers conversation context, and can manage your Gmail and Google Calendar on your behalf through MCP (Model Context Protocol) tool servers. It also has a dedicated sub-agent for general web/knowledge searches, and a dedicated error-handling workflow that alerts you by email when something breaks.

This repo contains **5 workflows**:

| # | Workflow | Role |
|---|----------|------|
| 1 | `Telegram Personal Assistant` | Main entry point — receives Telegram messages, runs the AI Agent, replies |
| 2 | `Gmail MCP Server` | Exposes Gmail actions as MCP tools |
| 3 | `Calendar MCP Server` | Exposes Google Calendar actions as MCP tools |
| 4 | `Telegram search tool` | Sub-agent workflow for web/knowledge lookups |
| 5 | `Error Workflow` | Catches failures from other workflows and emails a notification |

---

## Architecture

### 1. Telegram Personal Assistant (main workflow)

```
Telegram Trigger (message)
        │
        ▼
        If ──false──▶ Stop and Error
        │true
        ▼
      Switch (rules: text / audio)
   ┌────┴────┐
   │text     │audio
   ▼         ▼
Edit      Get a file (get: file)
Fields         │
   │            ▼
   │      Transcribe a recording
   │      (transcribe: audio)
   └────┬────┘
        ▼
      Merge (append)
        │
        ▼
      AI Agent ──▶ Send a text message (Telegram)
   ┌────┼──────────────────┬──────────────┬────────────────────┐
   ▼    ▼                  ▼              ▼                    ▼
Google  Simple            Calendar MCP   Gmail MCP    Call 'Telegram
Gemini  Memory            (tool)         (tool)        search tool'
Chat                                                    (tool)
Model
```

- **Telegram Trigger** — Listens for incoming Telegram messages.
- **If** — Validates the incoming update; invalid/unsupported updates are routed to **Stop and Error** (which is caught by the Error Workflow, see below).
- **Switch** — Branches on message type: plain **text** vs **audio** (voice notes).
- **Edit Fields** — Normalizes text messages into a consistent field for the agent.
- **Get a file** → **Transcribe a recording** — Downloads a voice message from Telegram and transcribes it to text.
- **Merge (append)** — Combines the text and transcribed-audio branches back into a single input stream for the agent.
- **AI Agent** — The core assistant. It:
  - Uses **Google Gemini Chat Model** as its LLM.
  - Uses **Simple Memory** to retain conversation context per chat.
  - Can call three tools:
    - **Calendar MCP** — create/read/update/delete Google Calendar events.
    - **Gmail MCP** — read, send, reply to, and manage Gmail messages.
    - **Call 'Telegram search tool'** — delegates general knowledge/search questions to the sub-agent workflow described below.
- **Send a text message** — Sends the agent's reply back to the user on Telegram.

### 2. Gmail MCP Server

```
MCP Server Trigger
        │
        ├── Send a message in Gmail
        ├── Delete a message in Gmail
        ├── Get many messages in Gmail
        ├── Mark a message as read in Gmail
        ├── Mark a message as unread in Gmail
        ├── Remove label from message in Gmail
        └── Reply to a message in Gmail
```

Exposes common Gmail operations as callable MCP tools. This is the server the main assistant's **Gmail MCP** tool connects to.

### 3. Calendar MCP Server

```
MCP Server Trigger
        │
        ├── Create an event in Google Calendar
        ├── Delete an event in Google Calendar
        ├── Get many events in Google Calendar
        ├── Update an event in Google Calendar
        └── Custom API call to Google Calendar
```

Exposes common Google Calendar operations as callable MCP tools. This is the server the main assistant's **Calendar MCP** tool connects to.

### 4. Telegram search tool (sub-agent)

```
When Executed by Another Workflow
        │
        ▼
      AI Agent
   ┌────┼──────────┬────────────────────┐
   ▼    ▼          ▼                    ▼
Google  (Memory)  Wikipedia    Get many items in
Gemini             tool         Hacker News (getAll: all)
Chat
Model
```

A standalone agent invoked by the main workflow whenever the user asks something that requires external lookup (general knowledge, current Hacker News stories, etc.). It runs its own Gemini-backed reasoning loop with **Wikipedia** and **Hacker News** as tools, then returns a result to the calling workflow.

### 5. Error Workflow

```
Error Trigger ──▶ Send a message (Gmail)
```

Set as the **Error Workflow** for the other workflows in n8n settings. Whenever any of them fails (e.g. the `Stop and Error` node fires, or a node throws), this workflow is triggered automatically and emails a failure notification via Gmail.

---

## Prerequisites

- An [n8n](https://n8n.io) instance (self-hosted or cloud)
- A [Telegram Bot](https://core.telegram.org/bots#how-do-i-create-a-bot) and its API token
- A [Google Gemini API key](https://ai.google.dev/) for the chat models
- A Google account with:
  - **Gmail API** enabled (OAuth2 credential)
  - **Google Calendar API** enabled (OAuth2 credential)
- Gmail credentials for the Error Workflow's notification email

---

## Setup

1. **Import all 5 workflows** into n8n (`Workflows → Import from File`).

2. **Configure credentials**
   - **Telegram API** — add your bot token to the Telegram Trigger and Send-message nodes.
   - **Google Gemini (PaLM) API** — add your Gemini API key to all Google Gemini Chat Model nodes (main agent + sub-agent).
   - **Gmail OAuth2** — connect your Gmail account in the Gmail MCP Server workflow and the Error Workflow.
   - **Google Calendar OAuth2** — connect your Google account in the Calendar MCP Server workflow.

3. **Publish/activate the MCP server workflows first**
   - Activate **Gmail MCP Server** and **Calendar MCP Server** so their MCP endpoints are live.
   - In the main **Telegram Personal Assistant** workflow, point the **Gmail MCP** and **Calendar MCP** tool nodes at these two MCP server endpoints.

4. **Wire up the sub-agent**
   - Ensure the **Call 'Telegram search tool'** tool node in the main workflow references the **Telegram search tool** workflow (via "Execute Workflow" / MCP call).

5. **Set the Error Workflow**
   - In each workflow's settings, set **Error Workflow** to `Error Workflow` so failures trigger an email alert.

6. **Activate the main workflow**
   - Turn on **Telegram Personal Assistant** so it starts listening for messages.

---

## Usage

- **Text a question** to your Telegram bot — the assistant replies using Gemini, with memory of the ongoing conversation.
- **Send a voice note** — it's transcribed automatically and handled exactly like a text message.
- **Ask calendar-related things** ("What's on my calendar tomorrow?", "Schedule a meeting at 3pm") — handled via the Calendar MCP tool.
- **Ask email-related things** ("Any unread emails from Alex?", "Reply to that email saying I'm in") — handled via the Gmail MCP tool.
- **Ask general knowledge questions** ("What's trending on Hacker News?", "Who is...") — delegated to the Telegram search sub-agent (Wikipedia / Hacker News).
- **If anything fails**, you'll get an email notification via the Error Workflow so you can debug quickly.

---

## Notes

- Swap Google Gemini for any other chat model n8n supports without changing the overall architecture.
- The Calendar MCP Server's tool node labels/operations don't all perfectly match their configured actions in the source screenshots (e.g., a node named "Delete an event" configured as `create: event`) — double check each tool's operation before relying on it in production.
- Simple Memory is session-based; swap in a persistent memory backend (e.g. Postgres/Redis) if you need history across restarts.
- Keep the MCP server workflows activated at all times — the main assistant depends on them being reachable.
