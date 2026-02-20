# Sutro Skill

A skill for AI agents working with [Sutro](https://sutro.email) — the programmable email client. Gives your agent the ability to read threads, draft and send emails, post comments, manage your inbox, and interact with your calendar.

Install this skill to bring full email context into your agent. Works with Claude Code, Cursor, OpenClaw, or any agent that reads markdown context.

## Installation

**Cursor** — Add this repo as a plugin source in Cursor settings, or copy `SKILL.md` directly into your project root.

**Claude Code** — Copy `SKILL.md` into your project root. Claude Code will pick it up automatically on the next session.

**OpenClaw** — Add the raw `SKILL.md` URL to your skill config, or clone this repo and point your `skills.load.extraDirs` at it.

## Repo Structure

```
sutro-skill/
├── SKILL.md
├── references/
│   ├── cli-commands.md
│   ├── api-endpoints.md
│   ├── webhook-events.md
│   ├── automations.md
│   ├── calendar.md
│   └── openclaw-integration.md
├── LICENSE
└── README.md
```

- **SKILL.md** — Required. Core principles, auth, and workflow. The entry point loaded by the agent.
- **references/** — Supporting documents loaded into context on demand, linked from `SKILL.md`.

## Contributing
1. Fork this repo
2. Make changes to `SKILL.md` or the files in `references/`
3. Submit a PR

## License
MIT — Copyright (c) 2026 Connor Sears
