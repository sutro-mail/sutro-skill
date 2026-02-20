# Sutro Calendar Reference

## Token Scopes

- `calendar_read` — View calendar events, availability, and pending invites
- `calendar_write` — Create, update, and delete events, respond to invites

## CLI Commands

Calendar CLI commands require a **full** account (not available on Lite).

```bash
sutro calendar --json              # List upcoming events
sutro calendar view <id> --json    # View event details
sutro calendar create --json       # Create event (interactive)
sutro calendar delete <id>         # Delete event
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/calendar/events` | List events |
| `GET` | `/calendar/events/:id` | View event |
| `POST` | `/calendar/events` | Create event |
| `PATCH` | `/calendar/events/:id` | Update event |
| `DELETE` | `/calendar/events/:id` | Delete event |
