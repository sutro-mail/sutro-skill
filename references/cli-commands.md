# Sutro CLI Commands Reference

Always use `--json` flag for structured output.

## Identity & Context

| Action | Command |
|--------|---------|
| Identity & unread counts | `sutro me --json` |
| Priority triage | `sutro focus --json` |
| Email-only triage | `sutro focus --domain email --json` |
| GitHub-only triage | `sutro focus --domain github --json` |
| Raw intents (debug) | `sutro intent --json` |

When the user asks "What should I focus on?", use `sutro focus --json` first. Use `sutro intent --json` only to explain *why* extraction happened.

## Inbox

| Action | Command |
|--------|---------|
| Primary inbox | `sutro inbox --limit 10 --json` |
| GitHub inbox | `sutro inbox github --json` |
| Other inbox | `sutro inbox other --json` |
| Custom inboxes | `sutro inbox list --json` |
| Custom inbox threads | `sutro inbox show <slug> --json` |

## Thread Operations

| Action | Command |
|--------|---------|
| List Gmail labels | `sutro label list --json` |
| View thread | `sutro thread view <token> --json` |
| Comment on thread | `sutro thread comment <token> "<msg>" --from-agent` |
| Archive | `sutro thread archive <token>` |
| Mark read | `sutro thread read <token>` |
| Add label | `sutro thread label <token> "<label>"` |
| Remove label | `sutro thread label <token> "<label>" --remove` |
| Trash | `sutro thread trash <token>` |
| Snooze | `sutro thread snooze <token> --until "<time>"` |
| Unsnooze | `sutro thread unsnooze <token>` |

Note: For `sutro thread comment`, you **must** pass `--from-agent` or the comment looks like it came from the user.
Snooze and unsnooze require a full Sutro account.

## Compose & Drafts

| Action | Command |
|--------|---------|
| Send email | `sutro compose --to "<email>" --subject "<subj>" --body "<body>" --send` |
| Create draft | `sutro compose --to "<email>" --subject "<subj>" --body "<body>" --json` |
| List drafts | `sutro draft list --json` |
| View draft | `sutro draft view <id> --json` |
| Edit draft | `sutro draft edit <id> --body "<body>" --json` |
| Send draft | `sutro draft send <id>` |

Routing behavior:
- Lite accounts route compose/draft write commands to Gmail pass-through endpoints (`/api/v1/gmail/drafts`, `/api/v1/gmail/messages/send`).
- Full accounts keep using Sutro inbox endpoints (`/api/v1/drafts`, `/api/v1/emails`).
- Draft IDs can be numeric (full) or Gmail string IDs (lite). Use the ID exactly as returned by `sutro draft list --json`.

## Search

```
sutro search "<query>" --json
```

### Search Filters

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

## Contacts

| Action | Command |
|--------|---------|
| (Contacts are accessed via search or thread viewing) | |

## Files

| Action | Command |
|--------|---------|
| View file | `sutro file view <id> --json` |
| Summarize PDF | `sutro file summarize <id> --json` |

## Conversations

| Action | Command |
|--------|---------|
| List | `sutro conversation list --json` |
| View | `sutro conversation view <id> --json` |
| Reply as agent | `sutro conversation reply <id> "<msg>"` |

Note: `sutro conversation reply` always posts as the agent — no `--from-agent` flag needed.

## Automations

| Action | Command |
|--------|---------|
| List automations | `sutro automation --json` |
| View automation | `sutro automation view <id> --json` |
| Create automation | `sutro automation create --from "..." --step archive` |
| Update automation | `sutro automation update <id> --step ...` |
| Delete automation | `sutro automation delete <id>` |

> **Note:** `/filters` still works as an alias for backward compatibility.

See the [automations reference](https://raw.githubusercontent.com/sutro-mail/sutro-skill/main/references/automations.md) for full details on steps, criteria, and examples.

## Knowledge Sources

| Action | Command |
|--------|---------|
| List KSs | `sutro ks --json` |
| View KS | `sutro ks view <id> --json` |
| Create KS | `sutro ks create --name "..." --json` |
| Delete KS | `sutro ks delete <id>` |
| Add source | `sutro ks add-source <id> --type github --repo owner/repo` |
| Remove source | `sutro ks remove-source <ks> <src>` |

### Add source

```bash
# GitHub repo (fetches .md/.mdx/.txt files from the folder)
sutro ks add-source 1 --type github --repo sutro-eng/docs --path help/articles

# Google Doc
sutro ks add-source 1 --type google-doc --url "https://docs.google.com/document/d/..."
```

## Focus

| Action | Command |
|--------|---------|
| Priority triage | `sutro focus --json` |
| Email-only triage | `sutro focus --domain email --json` |
| GitHub-only triage | `sutro focus --domain github --json` |

## Auth

Assume the user is authenticated. If you get an auth error, tell the user to run `sutro auth login`. Do not attempt to login yourself.

## Doctor & Status

| Action | Command |
|--------|---------|
| Check status | `sutro status --json` |
| Run diagnostics | `sutro doctor --json` |

## Account Types & Command Availability

`GET /me` returns `user.account_type` with one of:
- `"lite"`: API-only access with scoped tokens (inbox reads use Gmail pass-through endpoints)
- `"full"`: Full Sutro account access

| Category | Commands |
|----------|----------|
| Works on Lite | `me`, `status`, `doctor`, `auth`, `search`, `inbox`, `thread`, `compose`, `draft`, `calendar`, `label`, `version` |
| Requires Full | `focus`, `intent`, `automation` (`filter`), `agent`, `skill`, `ks` (`kb`), `workspace`, `contact`, `file`, `github`, `chat`, `conversation` |

Thread command availability on Lite is subcommand-specific. Core Gmail thread actions are supported via pass-through, but snooze/unsnooze are full-account only.

When a Lite account runs a full-account command, the CLI returns an upgrade message with the upgrade URL.
