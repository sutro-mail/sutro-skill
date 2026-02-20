# Sutro Webhook Events Reference

When Sutro sends a webhook, the payload tells you what the user wants. Three event types:

## `mention` — user @mentioned the agent in a thread

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

## `conversation_message` — user sent a chat message

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

## `email.received` — new email arrived (opt-in)

This event is **opt-in** and disabled by default. Enable it per-agent in Settings > Agents. Designed to be lightweight — useful for monitoring, triage, and routing workflows.

```json
{
  "event": "email.received",
  "thread_token": "abc123",
  "subject": "Invoice #1042",
  "from": "billing@acme.com",
  "preview": "Please find attached your invoice...",
  "timestamp": "2026-02-20T..."
}
```

**Respond by** reading the email with `sutro thread view <thread_token> --json`, then taking action only if warranted (e.g. error alerts, urgent requests). For most emails, no response is needed.

> **Cost note:** At high email volume, `email.received` can generate many webhook calls. Each call runs your agent and incurs API usage. Enable only when you have a specific automation use case.

## Webhook Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Webhook-Event` | Event type (`mention`, `conversation_message`, or `email.received`) |
| `X-Webhook-Signature` | HMAC-SHA256 signature of the request body |
| `User-Agent` | `Sutro-Webhook/1.0` |
| `Authorization` | `Bearer <auth_token>` (if configured on the endpoint) |
