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
| `GET` | `/calendar/availability` | Get available time slots |
| `POST` | `/calendar/events` | Create event |
| `PATCH` | `/calendar/events/:id` | Update event |
| `DELETE` | `/calendar/events/:id` | Delete event |
| `POST` | `/calendar/events/:id/rsvp` | Respond to invite |

### Create event body (`POST /calendar/events`)

```json
{
  "event": {
    "title": "Lunch meeting",
    "time": "2026-02-24T12:00:00-08:00",
    "duration_minutes": 60,
    "attendee_emails": ["alice@example.com", "bob@example.com"],
    "meeting_type": "video_call"
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | No | Event title (default: "Meeting") |
| `time` | **Yes** | ISO 8601 start time, e.g. `"2026-02-24T12:00:00-08:00"` |
| `duration_minutes` | No | Duration in minutes (default: 30) |
| `attendee_emails` | No | Array of email addresses to invite |
| `meeting_type` | No | `"video_call"` (default, adds Google Meet), `"phone_call"`, or `"in_person"` |
| `phone_number` | No | Required when `meeting_type` is `"phone_call"` |
| `location` | No | Location string for `"in_person"` meetings |

### Update event body (`PATCH /calendar/events/:id`)

```json
{
  "event": {
    "title": "Updated title",
    "new_time": "2026-02-25T14:00:00-08:00",
    "duration_minutes": 45,
    "add_attendees": ["charlie@example.com"],
    "remove_attendees": ["bob@example.com"],
    "location": "Conference Room B",
    "description": "Agenda: ..."
  }
}
```

All update fields are optional — only include what you want to change. To add attendees use `add_attendees`; to remove them use `remove_attendees`.

### RSVP body (`POST /calendar/events/:id/rsvp`)

```json
{ "response": "accepted" }
```

Valid values: `"accepted"`, `"declined"`, `"tentative"`.

### List events params (`GET /calendar/events`)

| Param | Description |
|-------|-------------|
| `start_date` | YYYY-MM-DD (default: today) |
| `end_date` | YYYY-MM-DD (default: today + 7 days) |
| `max_results` | Max events to return (default: 50, max: 100) |
