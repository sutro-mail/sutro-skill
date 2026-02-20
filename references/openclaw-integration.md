# Sutro + OpenClaw Integration Reference

If you're running as an external agent via OpenClaw (or n8n, custom webhook handler), these details are critical.

## OpenClaw Webhook Configuration

- Configure webhook hooks in `hooks.mappings` with `match.path`, `action: "agent"`, and a `messageTemplate`
- Use `{{field_name}}` to interpolate JSON payload fields (e.g., `{{conversation_id}}`, `{{body}}`, `{{thread_token}}`)
- **Gotcha:** `{{body}}` resolves to the JSON's `body` field, not the raw HTTP request body
- Set a `sessionKey` on the mapping so the agent maintains conversation history across webhook calls
- Set a model override to avoid using expensive models for routine webhook handling
- The agent needs the CLI on its PATH: `export PATH="/opt/homebrew/opt/node@22/bin:$PATH"` (or wherever the `sutro` binary is installed)

## Multi-Account Webhook Routing

Multiple Sutro accounts can send webhooks to the same endpoint (e.g. `/hooks/sutro`). To identify which account fired the webhook, include `{{user.email}}` or `{{sender_email}}` in your `messageTemplate`:

```
"messageTemplate": "[{{user.email}}] {{event}}: {{body}}"
```

The `user` object in every webhook payload contains the account owner's info, so `{{user.email}}` always resolves to the correct account. This is important when one agent serves multiple users (e.g. a shared AI assistant) — the agent can route actions to the right inbox or apply per-user logic based on the email or domain.

## Session Keys

Set a `sessionKey` on the mapping so the agent maintains conversation history across webhook calls. This allows the agent to remember context from previous interactions within the same conversation or thread.

Example mapping:
```json
{
  "match": { "path": "/hooks/sutro" },
  "action": "agent",
  "messageTemplate": "[{{user.email}}] {{event}}: {{body}}",
  "sessionKey": "sutro-{{conversation_id}}"
}
```
