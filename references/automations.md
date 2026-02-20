# Sutro Automations Reference

Automations automatically process incoming emails that match specific criteria or respond to external events. An automation is built from **workflow steps** — an ordered sequence of actions that run one after another:

- **Standard actions** — archive, trash, mark read, route to inbox, comment
- **AI skills** — draft, summarize, tasks, review, negotiate, cursor-fix, cursor-plan, linear-create-issue
- **Delays** — pause the workflow with optional cancel-on-interaction
- **Comments** — post a comment on the thread (supports @mentions to trigger agents)

Steps can be combined in any order.

## CLI Commands

| Action | Command |
|--------|---------|
| List automations | `sutro automation --json` |
| View automation | `sutro automation view <id> --json` |
| Create automation | `sutro automation create ...` |
| Update automation | `sutro automation update <id> ...` |
| Delete automation | `sutro automation delete <id>` |

> **Note:** `/filters` still works as an alias for backward compatibility.

## Create Automation (CLI)

**Workflow steps (recommended) — use `--step` flag (repeatable):**

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

### Step Values

`archive`, `trash`, `read`, `label:<name>`, `inbox:<slug>`, `"comment:<text>"`, `delay:<duration>[:if-unread]`, `<skill>`, `<skill>:prompt:<text>`, `<skill>:ks:<name>`, `<skill>:prompt:<text>:ks:<name>`.

### Criteria Flags

`--from`, `--to`, `--subject`, `--query`, `--github-tab`, `--attachment`, `--event`. At least one required (event-triggered automations don't need email criteria).

### Legacy Flags (still supported when `--step` is not used)

```bash
sutro automation create --from "notifications@github.com" --action archive
sutro automation create --from "support@" --sutro-action draft --knowledge-source "Support docs"
sutro automation create --from "@marketing.com" --action trash --delay 2d --cancel-on-interaction
```

### Event Triggers

`--event` values: `pr-merged`, `pr-closed`, `issue-closed`, `new-reply`, `user-sent-reply`, `meeting-scheduled`, `github-ci-passed`, `github-review-requested`, `linear-issue-completed`, `intent:recruiter`, `intent:approval-request`, `intent:deadline`, `intent:intro-made`, `intent:schedule-meeting`, `intent:signature-request`, `intent:action-request`, `intent:question`, `intent:commitment`, `intent:followup`, `intent:fyi`.

These can optionally include email criteria to narrow which threads they apply to. Intent triggers fire only when Sutro detects the intent with high confidence.

### Delay Options

`--delay 1h|6h|1d|2d|3d|1w` sets a grace period before the standard action fires. Add `--cancel-on-interaction` to cancel the delayed action if the email is read before the delay expires. Delay applies to the standard action only — Sutro actions always fire immediately.

## Update Automation (CLI)

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

## Create Automation (API)

**Steps format (recommended):**

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

### Step Types

`archive`, `trash`, `mark_read`, `label` (config: `label_name`), `route_to_inbox` (config: `inbox_id`), `comment` (config: `body`), `delay` (config: `minutes`, `cancel_on_interaction`), `custom_action` (config: `slug`, optional `sutro_prompt`, `knowledge_source_id`).

**Legacy format (still supported when `steps` is not provided):**

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

### Criteria Fields

`from` (email or `@domain`), `to`, `subject` (contains), `query` (Gmail search syntax), `github_tab` (`"for_you"` or `"everything_else"`), `has_attachment: true`.

### Event Trigger Field

`event_trigger`: `"pr_merged"`, `"pr_closed"`, `"issue_closed"`, `"new_reply"`, `"user_sent_reply"`, `"meeting_scheduled"`, `"github_ci_passed"`, `"github_review_requested"`, `"linear_issue_completed"`, `"intent:recruiter"`, `"intent:approval_request"`, `"intent:deadline"`, `"intent:intro_made"`, `"intent:schedule_meeting"`, `"intent:signature_request"`, `"intent:action_request"`, `"intent:question"`, `"intent:commitment"`, `"intent:followup"`, `"intent:fyi"`, or `null`.

Response includes `matching_existing_count`, `apply_to_existing_queued`, and `already_exists` (true when an identical automation was reused instead of creating a duplicate).

## Update Automation (API)

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
