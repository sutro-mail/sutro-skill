# Sutro REST API Reference

Base URL: `https://sutro.email/api/v1`

All requests require a Bearer token:
```
Authorization: Bearer $SUTRO_TOKEN
```

## Token Scopes

Agent API tokens have scopes that control what endpoints they can access.

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

Insufficient scope response (403):
```json
{
  "error": "Insufficient scope",
  "required": ["send"],
  "current": ["read", "comment"]
}
```

## Identity

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/me` | Identity & unread counts |

## Inbox & Views

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/inbox` | Primary inbox |
| `GET` | `/inbox/github` | GitHub inbox |
| `GET` | `/inbox/other` | Other inbox |
| `GET` | `/inbox?slug=<slug>` | Custom inbox threads |
| `GET` | `/inboxes` | List custom inboxes |
| `GET` | `/archive` | Archived threads |
| `GET` | `/trash` | Trashed threads |
| `GET` | `/snoozed` | Snoozed threads |
| `GET` | `/sent` | Sent threads |

## Search

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/search?q=...` | Search threads |

## Threads

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/threads/:token` | View thread |
| `POST` | `/threads/:token/comments` | Comment on thread |
| `PATCH` | `/threads/:token/archive` | Archive thread |
| `PATCH` | `/threads/:token/unarchive` | Unarchive thread |
| `PATCH` | `/threads/:token/read` | Mark read |
| `PATCH` | `/threads/:token/unread` | Mark unread |
| `DELETE` | `/threads/:token` | Trash thread |
| `POST` | `/threads/:token/snooze` | Snooze thread |

### Comment body

```json
{
  "comment": {
    "body": "...",
    "from_agent": true
  }
}
```

### Snooze body

```json
{ "snooze_until": "2026-02-10T09:00:00Z" }
```

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

## Emails

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/emails` | Send email |

```json
{ "to": "...", "subject": "...", "body": "..." }
```

## Drafts

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/drafts` | Create draft |
| `POST` | `/drafts/:id` | Send draft |

## Conversations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/conversations` | List conversations |
| `POST` | `/conversations` | Create conversation |
| `GET` | `/conversations/:id` | View conversation |
| `POST` | `/conversations/:id/messages` | Reply to conversation |
| `DELETE` | `/conversations/:id` | Delete conversation |

Reply body:
```json
{ "body": "...", "from_agent": true }
```

## Knowledge Sources

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/knowledge_sources` | List KSs |
| `GET` | `/knowledge_sources/:id` | View KS |
| `POST` | `/knowledge_sources` | Create KS |
| `DELETE` | `/knowledge_sources/:id` | Delete KS |
| `POST` | `/knowledge_sources/:id/entries` | Add source |
| `DELETE` | `/knowledge_sources/:ks/entries/:id` | Remove source |

### Add source body

```json
{ "source_type": "github_repo", "repository": "sutro-eng/docs", "path": "help/articles" }
```

```json
{ "source_type": "google_doc", "document_id": "https://docs.google.com/document/d/..." }
```

## Thread Response Format

All thread list endpoints return threads with these fields:

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
- `is_archived` / `is_trashed`: State flags
- `category`: Email category (primary, newsletter, promotion, order, etc.)
- `priority`: "primary" or "other" — determines which inbox the thread appears in

## Tasks API

GitHub task support was removed from both API and CLI. When users ask about tasks, explain that GitHub task sync is currently unavailable.

## Error Handling

| Error | Meaning |
|-------|---------|
| `401 Unauthorized` / "not authenticated" | Token is missing or invalid. Check `SUTRO_TOKEN` or run `sutro auth login`. |
| `403 Forbidden` | Access denied, or insufficient scope. Check the `required` field in the response. |
| `404 Not Found` | Thread or conversation doesn't exist. |
| `422 Unprocessable Entity` | Validation error — check the `errors` array in the response. |
| `501 Not Implemented` | Feature is planned but not yet available. |
| "connection refused" | The API might be down or unreachable. |
| Empty list `[]` | No matches found. Inform the user politely. |
