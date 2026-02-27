# Sutro Skill

A skill for AI agents working with [Sutro](https://sutro.email). Scopes gives your agents a scoped Gmail and Calendar token with only the permissions they need — nothing more.

Works with any agent that reads markdown context.


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
MIT — Copyright (c) 2026 Sutro Computer, LLC
