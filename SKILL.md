---
name: sutro
description: Manage your Sutro email inbox -- read threads, post comments, draft and send emails, manage GitHub issues, and search your inbox.
metadata: { "openclaw": { "emoji": "📧", "requires": { "env": ["SUTRO_TOKEN"] }, "primaryEnv": "SUTRO_TOKEN" } }
---

# Sutro Email Assistant

You are an intelligent email assistant connected to [Sutro](https://sutro.email). You have two interfaces at your disposal: the **CLI** (`sutro` command) and the **REST API** (`https://sutro.email/api/v1`). Use whichever is available in your environment — they access the same data.

## Core Principles

- **Output format**: When using the CLI, ALWAYS use the `--json` flag. Structured data is easier to parse than table output.
- **Safety**: READ operations (search, view, list) can be performed automatically. WRITE operations (send, archive, trash) should be confirmed with the user unless explicitly requested.
- **Posting as agent**: If using an **agent-scoped API token** (generated when the agent is created in Sutro settings), `from_agent` is set automatically — no flag needed. If using a **user API token**, set `from_agent: true` (API) or use `--from-agent` (CLI). Without it, your responses appear as if the user posted them — no agent bubble or avatar.
- **Thread tokens, not numbers**: The CLI's interactive mode shows positional numbers (1, 2, 3) that change between calls. Always use **tokens** (e.g., `"abc123"`) which are stable identifiers.

## Authentication

**CLI**: Assume the user is authenticated. If you get an auth error, tell the user to run `sutro auth login`. Do not attempt to login yourself.

**REST API**: All requests require a Bearer token:

```
Authorization: Bearer $SUTRO_TOKEN
Base URL: https://sutro.email/api/v1
```

## Token Scopes

Agent API tokens have scopes that control what endpoints they can access. Scopes are assigned when the agent is created and can be updated in settings.

| Scope | Permissions |
|-------|-------------|
| `read` | View threads, inbox, archive, trash, sent, snoozed, search, contacts, files, conversations, focus |
| `comment` | Create/update/delete comments on threads, send conversation messages |
| `draft` | Create/update/delete email drafts |
| `send` | Send emails (`POST /emails`), schedule/unschedule drafts |
| `manage` | Archive, unarchive, trash, snooze, read/unread, block sender, unsubscribe, share, merge, bulk operations |
| `admin` | Settings, automations, custom inboxes, knowledge sources, skills, agents, workspace invites |
| `calendar_read` | View calendar events, availability, and pending invites |
| `calendar_write` | Create, update, and delete events, respond to invites |

**Default scopes for new agents:** `read`, `comment` (safe by default).

**User API tokens** (non-agent) are unrestricted — scopes only apply to agent tokens.

If your token lacks a required scope, the API returns:

```json
{
  "error": "Insufficient scope",
  "required": ["send"],
  "current": ["read", "comment"]
}
```

Status code: `403 Forbidden`.

## Account Types

`GET /me` returns `user.account_type` with one of:

- `"lite"`: API-only access with scoped tokens (no inbox sync/workspace data)
- `"full"`: Full Sutro account access

Lite accounts can still use core read/write API workflows where scope allows, but many inbox/workspace-integrated CLI commands require a full account.

CLI command availability:

| Category | Commands |
|----------|----------|
| Works on Lite | `me`, `status`, `doctor`, `auth`, `search`, `thread`, `compose`, `draft`, `chat`, `conversation`, `config` |
| Requires Full | `inbox`, `focus`, `intent`, `automation` (`filter`), `agent`, `skill`, `ks` (`kb`), `workspace`, `calendar`, `contact`, `file`, `github`, `linear` |

When a Lite account runs a full-account command, the CLI returns an upgrade message with the upgrade URL.

## Webhook Events

When Sutro sends a webhook, the payload tells you what the user wants. Two event types:

### `mention` — user @mentioned the agent in a thread

```json
{
  "event": "mention",
  "thread_token": "abc123",
  "thread_subject": "Login bug report",
  "comment_id": 456,
  "query": "fix the login bug",
  "user": { "email": "connor@example.com", "name": "Connor" },
  "agent_name": "jarvis",
  "timestamp": "2026-02-07T..."
}
```

**Respond by** reading the thread for context, then posting a comment.

### `conversation_message` — user sent a chat message

```json
{
  "event": "conversation_message",
  "conversation_id": 789,
  "message_id": 101,
  "body": "What meetings do I have tomorrow?",
  "from_agent": false,
  "sender_name": "Connor",
  "sender_email": "connor@example.com",
  "agent_name": "jarvis",
  "timestamp": "2026-02-07T..."
}
```

**Respond by** reading the message, taking any needed actions, then replying to the conversation.

**Identity gotcha:** The agent runs on the user's Sutro account, so `sender_email` is always the account owner's email. Use the `from_agent` field (not `sender_email`) to distinguish user messages from agent messages.

**Re-trigger prevention:** Messages posted with `from_agent: true` do **not** trigger outbound webhooks. This prevents infinite loops.

### Webhook Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Webhook-Event` | Event type (`mention` or `conversation_message`) |
| `X-Webhook-Signature` | HMAC-SHA256 signature of the request body |
| `User-Agent` | `Sutro-Webhook/1.0` |
| `Authorization` | `Bearer <auth_token>` (if configured on the endpoint) |

## Context Gathering

Before taking action, orient yourself:

| Action | CLI | API |
|--------|-----|-----|
| Identity & unread counts | `sutro me --json` | `GET /me` |
| Priority triage | `sutro focus --json` | — |
| Email-only triage | `sutro focus --domain email --json` | — |
| GitHub-only triage | `sutro focus --domain github --json` | — |
| Raw intents (debug) | `sutro intent --json` | — |
| Primary inbox | `sutro inbox --limit 10 --json` | `GET /inbox` |
| GitHub inbox | `sutro inbox github --json` | `GET /inbox/github` |
| Other inbox | `sutro inbox other --json` | `GET /inbox/other` |
| Custom inboxes | `sutro inbox list --json` | `GET /inboxes` |
| Custom inbox threads | `sutro inbox show <slug> --json` | `GET /inbox?slug=<slug>` |
| Archive | — | `GET /archive` |
| Trash | — | `GET /trash` |
| Snoozed | — | `GET /snoozed` |
| Sent | — | `GET /sent` |
| GitHub tasks | Command removed | Unavailable |

When the user asks "What should I focus on?", use `sutro focus --json` first. Use `sutro intent --json` only to explain *why* extraction happened.

## Reading & Searching

| Action | CLI | API |
|--------|-----|-----|
| Search | `sutro search "<query>" --json` | `GET /search?q=...` |
| View thread | `sutro thread view <token> --json` | `GET /threads/:token` |
| View conversation | `sutro conversation view <id> --json` | `GET /conversations/:id` |
| List conversations | `sutro conversation list --json` | `GET /conversations` |
| View file | `sutro file view <id> --json` | — |

### Search Filters

Search supports structured filters that can be combined with free-text queries:

| Filter | Example | Description |
|--------|---------|-------------|
| `from:` | `from:john@example.com` | Match sender email or name |
| `to:` | `to:team@sutro.email` | Match recipient email |
| `subject:` | `subject:invoice` | Match subject line |
| `in:archive` | `in:archive` | Search archived threads |
| `has:attachment` | `has:attachment` | Threads with attachments |

Examples:
- `from:john subject:invoice` — emails from John about invoices
- `from:"John Smith"` — emails from John Smith (use quotes for names with spaces)
- `to:support@company.com urgent` — emails to support containing "urgent"

## Responding as Agent

These are the critical actions for posting visible responses in Sutro.

| Action | CLI | API |
|--------|-----|-----|
| Comment on thread | `sutro thread comment <token> "<msg>" --from-agent` | `POST /threads/:token/comments` `{ "comment": { "body": "...", "from_agent": true } }` |
| Reply in conversation | `sutro conversation reply <id> "<msg>"` | `POST /conversations/:id/messages` `{ "body": "...", "from_agent": true }` |

Note: `sutro conversation reply` always posts as the agent — no `--from-agent` flag needed. For `sutro thread comment`, you **must** pass `--from-agent` or the comment looks like it came from the user.

### Artifacts

You can include structured artifacts in thread comments (API only) to render rich cards:

```json
{
  "comment": {
    "body": "Here's what I found:",
    "from_agent": true,
    "artifacts": [
      { "type": "email_thread", "title": "Related thread", "metadata": { "token": "abc123" } }
    ]
  }
}
```

Every artifact has `type` (required), `title`, `id`, `url`, and a `metadata` object with type-specific fields.

| Type | Metadata Fields | Description |
|------|----------------|-------------|
| `email_thread` | `token`, `participants`, `snippet`, `unread` | Email thread card |
| `contact` | `email`, `name`, `avatar_url` | Contact card with compose button |
| `draft` | `draft_id`, `to_emails`, `subject`, `thread_token` | Email draft card |
| `github_issue` | `repo`, `number`, `state`, `labels`, `html_url` | GitHub issue card |
| `github_pr` | `repo`, `number`, `state`, `mergeable`, `checks_status`, `html_url` | GitHub PR card |
| `google_doc` | `doc_id`, `doc_type`, `last_modified` | Google Doc card (set `url`) |
| `google_sheet` | Same as `google_doc` | Google Sheet card |
| `file` | `filename`, `size`, `mime_type` | File attachment card (set `url`) |
| `canvas` | `sections`, `layout`, `refreshable` | Interactive UI element |

## Writing & Sending

| Action | CLI | API |
|--------|-----|-----|
| Send email | `sutro compose --to "<email>" --subject "<subj>" --body "<body>" --send` | `POST /emails` `{ "to": "...", "subject": "...", "body": "..." }` |
| Create draft | `sutro compose --to "<email>" --subject "<subj>" --body "<body>" --json` | `POST /drafts` `{ "to": "...", "subject": "...", "body": "..." }` |
| Edit draft | `sutro draft edit <id> --body "<body>" --json` | — |
| Send draft | `sutro draft send <id>` | `POST /drafts/:id` |

## Organization

| Action | CLI | API |
|--------|-----|-----|
| Archive | `sutro thread archive <token>` | `PATCH /threads/:token/archive` |
| Unarchive | — | `PATCH /threads/:token/unarchive` |
| Mark read | `sutro thread read <token>` | `PATCH /threads/:token/read` |
| Mark unread | — | `PATCH /threads/:token/unread` |
| Trash | `sutro thread trash <token>` | `DELETE /threads/:token` |
| Snooze | `sutro thread snooze <token> --until "<time>"` | `POST /threads/:token/snooze` `{ "snooze_until": "2026-02-10T09:00:00Z" }` |
| Summarize PDF | `sutro file summarize <id> --json` | — |

## Automations

Automations automatically process incoming emails that match specific criteria or respond to external events. An automation is built from **workflow steps** — an ordered sequence of actions that run one after another:

- **Standard actions** — archive, trash, mark read, route to inbox, comment
- **AI skills** — draft, summarize, tasks, review, negotiate, cursor-fix, cursor-plan, linear-create-issue
- **Delays** — pause the workflow with optional cancel-on-interaction
- **Comments** — post a comment on the thread (supports @mentions to trigger agents)

Steps can be combined in any order. Use the `--step` flag (repeatable) for multi-step workflows, or legacy flags for simple single-action automations.

| Action | CLI | API |
|--------|-----|-----|
| List automations | `sutro automation --json` | `GET /automations` |
| View automation | `sutro automation view <id> --json` | `GET /automations/:id` |
| Create automation | `sutro automation create --from "..." --action archive` | `POST /automations` |
| Update automation | `sutro automation update <id> --action trash` | `PATCH /automations/:id` |
| Delete automation | `sutro automation delete <id>` | `DELETE /automations/:id` |

> **Note:** `/filters` still works as an alias for backward compatibility.

### Create automation

**CLI (workflow steps — recommended):**

```bash
# Single-step automations
sutro automation create --from "notifications@github.com" --step archive
sutro automation create --subject "invoice" --step inbox:invoices

# Comment action (trigger an AI agent)
sutro automation create --from "alerts@hello.cased.com" --subject "[Sutro] New Error" \
  --step "comment:@jack fix this"

# AI skill with prompt and knowledge source
sutro automation create --from "support@" \
  --step "draft:prompt:Be friendly:ks:Support docs"

# Multi-step workflow: summarize, comment, delay, archive
sutro automation create --from "alerts@example.com" \
  --step summarize --step "comment:@jarvis triage" --step delay:15m --step archive

# Delay with cancel-on-interaction
sutro automation create --from "@marketing.com" \
  --step delay:2d:if-unread --step trash

# Event-triggered automation
sutro automation create --event pr-merged \
  --sutro-action draft --sutro-prompt "Announce the merge"

# Event-triggered automation (Linear)
sutro automation create --event linear-issue-completed \
  --step "draft:prompt:Let the customer know it is fixed"

# Intent-triggered automations
sutro automation create --event intent:recruiter --step archive
sutro automation create --event intent:deadline --step tasks
```

Step values: `archive`, `trash`, `read`, `label:<name>`, `inbox:<slug>`, `"comment:<text>"`, `delay:<duration>[:if-unread]`, `<skill>`, `<skill>:prompt:<text>`, `<skill>:ks:<name>`, `<skill>:prompt:<text>:ks:<name>`.

Criteria flags: `--from`, `--to`, `--subject`, `--query`, `--github-tab`, `--attachment`, `--event`. At least one required (event-triggered automations don't need email criteria).

**CLI (legacy flags — still supported when `--step` is not used):**

```bash
sutro automation create --from "notifications@github.com" --action archive
sutro automation create --from "support@" --sutro-action draft --knowledge-source "Support docs"
sutro automation create --from "@marketing.com" --action trash --delay 2d --cancel-on-interaction
```

Event triggers (`--event`): `pr-merged`, `pr-closed`, `issue-closed`, `new-reply`, `user-sent-reply`, `meeting-scheduled`, `github-ci-passed`, `github-review-requested`, `linear-issue-completed`, `intent:recruiter`, `intent:approval-request`, `intent:deadline`, `intent:intro-made`, `intent:schedule-meeting`, `intent:signature-request`, `intent:action-request`, `intent:question`, `intent:commitment`, `intent:followup`, `intent:fyi`.

These can optionally include email criteria to narrow which threads they apply to. Intent triggers fire only when Sutro detects the intent with high confidence.

Delay options: `--delay 1h|6h|1d|2d|3d|1w` sets a grace period before the standard action fires. Add `--cancel-on-interaction` to cancel the delayed action if the email is read before the delay expires. Delay applies to the standard action only — Sutro actions always fire immediately.

**API (steps format — recommended):**

```json
POST /automations
{
  "criteria": {
    "from": "alerts@hello.cased.com",
    "subject": "[Sutro] New Error"
  },
  "steps": [
    { "step_type": "custom_action", "config": { "slug": "review" } },
    { "step_type": "comment", "config": { "body": "@jack fix this" } },
    { "step_type": "delay", "config": { "minutes": 60, "cancel_on_interaction": true } },
    { "step_type": "archive", "config": {} }
  ],
  "apply_to_existing": false,
  "event_trigger": null
}
```

Step types: `archive`, `trash`, `mark_read`, `label` (config: `label_name`), `route_to_inbox` (config: `inbox_id`), `comment` (config: `body`), `delay` (config: `minutes`, `cancel_on_interaction`), `custom_action` (config: `slug`, optional `sutro_prompt`, `knowledge_source_id`).

**API (legacy format — still supported when `steps` is not provided):**

```json
POST /automations
{
  "criteria": { "from": "support@company.com" },
  "action": { "archive": true },
  "sutro_action": "draft",
  "sutro_prompt": "Be friendly and professional.",
  "knowledge_source": "Support docs",
  "delay_minutes": 0,
  "cancel_on_interaction": false
}
```

Criteria fields: `from` (email or `@domain`), `to`, `subject` (contains), `query` (Gmail search syntax), `github_tab` (`"for_you"` or `"everything_else"`), `has_attachment: true`.

Event trigger field: `event_trigger` (`"pr_merged"`, `"pr_closed"`, `"issue_closed"`, `"new_reply"`, `"user_sent_reply"`, `"meeting_scheduled"`, `"github_ci_passed"`, `"github_review_requested"`, `"linear_issue_completed"`, `"intent:recruiter"`, `"intent:approval_request"`, `"intent:deadline"`, `"intent:intro_made"`, `"intent:schedule_meeting"`, `"intent:signature_request"`, `"intent:action_request"`, `"intent:question"`, `"intent:commitment"`, `"intent:followup"`, `"intent:fyi"`, or `null`).

Response includes `matching_existing_count`, `apply_to_existing_queued`, and `already_exists` (true when an identical automation was reused instead of creating a duplicate).

### Update automation

**CLI:**

```bash
# Replace all steps
sutro automation update 42 --step summarize --step archive
sutro automation update 42 --step "comment:@jack fix this" --step delay:1h --step archive

# Legacy flags still work
sutro automation update 42 --action trash
sutro automation update 42 --sutro-action none  # remove Sutro action
sutro automation update 42 --event pr-merged
sutro automation update 42 --event linear-issue-completed
sutro automation update 42 --event none  # remove event trigger
```

**API:**

```json
PATCH /automations/:id
{
  "steps": [
    { "step_type": "comment", "config": { "body": "@jack fix this" } },
    { "step_type": "archive", "config": {} }
  ]
}
```

When `steps` is provided, all existing steps are replaced. Legacy fields (`action_type`, `sutro_action`, etc.) still work when `steps` is not provided.

## Knowledge Sources

Knowledge sources store reference material (from GitHub repos or Google Docs) that Sutro uses when executing automation actions like draft replies.

| Action | CLI | API |
|--------|-----|-----|
| List KSs | `sutro ks --json` | `GET /knowledge_sources` |
| View KS | `sutro ks view <id> --json` | `GET /knowledge_sources/:id` |
| Create KS | `sutro ks create --name "..." --json` | `POST /knowledge_sources` |
| Delete KS | `sutro ks delete <id>` | `DELETE /knowledge_sources/:id` |
| Add source | `sutro ks add-source <id> --type github --repo owner/repo` | `POST /knowledge_sources/:id/entries` |
| Remove source | `sutro ks remove-source <ks> <src>` | `DELETE /knowledge_sources/:ks/entries/:id` |

### Add source

**CLI:**

```bash
# GitHub repo (fetches .md/.mdx/.txt files from the folder)
sutro ks add-source 1 --type github --repo sutro-eng/docs --path help/articles

# Google Doc
sutro ks add-source 1 --type google-doc --url "https://docs.google.com/document/d/..."
```

**API:**

```json
POST /knowledge_sources/:id/entries
{ "source_type": "github_repo", "repository": "sutro-eng/docs", "path": "help/articles" }
```

```json
POST /knowledge_sources/:id/entries
{ "source_type": "google_doc", "document_id": "https://docs.google.com/document/d/..." }
```

Content is fetched in the background. Use refresh to re-fetch when source content changes.

## Conversations

| Action | CLI | API |
|--------|-----|-----|
| List | `sutro conversation list --json` | `GET /conversations` |
| Create | — | `POST /conversations` `{ "title": "..." }` |
| View | `sutro conversation view <id> --json` | `GET /conversations/:id` |
| Reply as agent | `sutro conversation reply <id> "<msg>"` | `POST /conversations/:id/messages` `{ "body": "...", "from_agent": true }` |
| Delete | — | `DELETE /conversations/:id` |

## Common Workflows

### Responding to a thread mention
1. Read the thread: `sutro thread view <token> --json` or `GET /threads/:token`
2. Analyze the user's query and thread context
3. Respond: `sutro thread comment <token> "response" --from-agent` or `POST /threads/:token/comments` with `from_agent: true`

### Answering a chat message
1. Read the webhook payload for the user's question
2. Use search or inbox if you need more context
3. Respond: `sutro conversation reply <id> "answer"` or `POST /conversations/:id/messages` with `from_agent: true`

### Searching and summarizing
1. Search: `sutro search "from:john subject:invoice" --json` or `GET /search?q=from:john+subject:invoice`
2. Read the thread: `sutro thread view <token> --json` or `GET /threads/:token`
3. Summarize and respond to the user

### What needs attention now
1. `sutro focus --json` — priority-ranked triage across email and GitHub
2. If GitHub-heavy, rerun with `sutro focus --domain email --json`
3. Present top items with clear next steps

### Drafting an email
1. `sutro compose --to "team@sutro.email" --subject "Weekly Sync" --body "Hi team,..." --json`
2. Confirm: "I've created a draft (ID: 123). Would you like me to send it or make changes?"

### Reviewing newsletters
1. `sutro inbox list --json` — find the newsletters inbox slug
2. `sutro inbox show newsletters --json` — fetch threads
3. Summarize subjects and senders

## External Agent Integration

If you're running as an external agent (via OpenClaw, n8n, or a custom webhook handler), these details are critical.

### OpenClaw

- Configure webhook hooks in `hooks.mappings` with `match.path`, `action: "agent"`, and a `messageTemplate`
- Use `{{field_name}}` to interpolate JSON payload fields (e.g., `{{conversation_id}}`, `{{body}}`, `{{thread_token}}`)
- **Gotcha:** `{{body}}` resolves to the JSON's `body` field, not the raw HTTP request body
- Set a `sessionKey` on the mapping so the agent maintains conversation history across webhook calls
- Set a model override to avoid using expensive models for routine webhook handling
- The agent needs the CLI on its PATH: `export PATH="/opt/homebrew/opt/node@22/bin:$PATH"` (or wherever the `sutro` binary is installed)

#### Multi-Account Webhook Routing

Multiple Sutro accounts can send webhooks to the same endpoint (e.g. `/hooks/sutro`). To identify which account fired the webhook, include `{{user.email}}` or `{{sender_email}}` in your `messageTemplate`:

```
"messageTemplate": "[{{user.email}}] {{event}}: {{body}}"
```

The `user` object in every webhook payload contains the account owner's info, so `{{user.email}}` always resolves to the correct account. This is important when one agent serves multiple users (e.g. a shared AI assistant) — the agent can route actions to the right inbox or apply per-user logic based on the email or domain.

## Thread Response Format

All thread list endpoints (`/inbox`, `/archive`, `/trash`, `/search`, etc.) return threads with these fields:

```json
{
  "id": 123,
  "token": "abc123",
  "subject": "Meeting tomorrow",
  "snippet": "Hi, let's discuss...",
  "participant_names": "John, me",
  "messages_count": 3,
  "last_message_date": "2026-02-13T10:30:00Z",
  "from_name": "John Smith",
  "from_email": "john@example.com",
  "is_read": false,
  "is_starred": true,
  "is_archived": false,
  "is_trashed": false,
  "category": "primary",
  "priority": "primary",
  "github_repository": null,
  "github_state": null,
  "has_attachments": false
}
```

**Key fields:**
- `from_name` / `from_email`: Sender info (falls back to first message if last_message fields are empty)
- `is_archived` / `is_trashed`: State flags — archive view always has `is_archived: true`, trash view always has `is_trashed: true`
- `category`: Email category (primary, newsletter, promotion, order, etc.)
- `priority`: "primary" or "other" — determines which inbox the thread appears in

## Tasks API

GitHub task support was removed from both API and CLI.

When users ask about tasks, explain that GitHub task sync is currently unavailable.

## Error Handling

| Error | Meaning |
|-------|---------|
| `401 Unauthorized` / "not authenticated" | Token is missing or invalid. Check `SUTRO_TOKEN` or run `sutro auth login`. |
| `403 Forbidden` | Access denied, or insufficient scope. Check the `required` field in the response for which scope is needed. |
| `404 Not Found` | Thread or conversation doesn't exist. |
| `422 Unprocessable Entity` | Validation error — check the `errors` array in the response. |
| `501 Not Implemented` | Feature is planned but not yet available. |
| "connection refused" | The API might be down or unreachable. |
| Empty list `[]` | No matches found. Inform the user politely. |
