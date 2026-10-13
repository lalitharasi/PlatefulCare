# Plateful Care (with SQLite)

Run it (Python 3.8+, no installs needed):

    python server.py

Open http://127.0.0.1:8000 (not the file itself). The first run creates `care.db` and loads the sample tickets.
Tickets, the group, status changes, ratings and the history log now survive a page refresh.
Delete `care.db` to reset.

## What is stored (SQLite)
- `tickets`, `issues`, `ticket_events` (written automatically on every stage change)
- `ticket_photos` (the customer's photos, shrunk in the browser) and `ticket_messages` (support and customer chat)
- Checks in the database: valid mood, intent, stage 0 to 4, rating 0 to 5, subject 1 to 200 characters

## API
| Method and path | Purpose |
|---|---|
| GET /api/state | all tickets and groups |
| PUT /api/tickets/{id} | create or update a ticket (validated) |
| PUT /api/issues/{id} | open or close a group |
| GET /api/tickets/{id}/events | status history |
| DELETE /api/tickets/{id} | delete a report with its photos, messages and history (support: any report; customer: own reports only) |
| GET /api/search?q=text | search reports by subject text or report number (customers only see their own) |
| POST /api/tickets/{id}/photos | add a photo (base64 JPEG, PNG or WebP, under 3 MB, up to 10 per report) |
| GET /api/tickets/{id}/photos, GET /api/photos/{id} | list and fetch photos (support only) |
| POST /api/tickets/{id}/messages | send a chat message (support or the report's customer) |
| GET /api/messages?after=N | new chat messages since id N (customers see only their own reports) |
| POST /api/login | sign in with `email`, `password`, `role` (`customer` or `support`) |
| POST /api/logout | sign out |
| GET /api/me | the signed-in user |

All routes except login and logout need a signed-in session (cookie). Only support accounts can change a group.

Errors use one format: `{ "code", "message", "field" }`.

## Not included
Reports are not tied to an individual customer account (a report is simply marked as the customer's own).
Without the server, `index.html` still works in memory only.

## Photos and "Talk to customer"
Customers' photos are uploaded when a report is sent and stored in the database. Support sees a **View photos** button on a report that has photos.
**Talk to customer** opens a chat for that report. The customer's page checks for new messages every 3 seconds, shows a notice and opens the **Talk to your support** chat with the message. Their replies go back to the agent. To try it, sign in as the customer in one browser and as support in another (or a private window), because the login cookie is shared inside one browser.

## Login
The page asks you to sign in as a Customer or a Support agent. Two demo accounts are created on the first run:

| Role | Email | Password |
|---|---|---|
| Customer | customer@plateful.test | customer123 |
| Support agent | support@plateful.test | support123 |

Passwords are stored salted and hashed (PBKDF2) in the `users` table, and sessions last 8 hours in an HttpOnly cookie. Five wrong passwords lock that email for a minute. Change the demo passwords before real use. Without the server there are no accounts, so the memory-only page skips login.

## Support chat bot (small LLM)
The chat window calls `POST /api/chat`. The server adds facts about the selected report (subject, stage, expected update) and asks a small model for a short reply. If the model is not available it falls back to the scripted replies, so the chat never breaks. Watch the terminal: it prints why it fell back.

Option A, free and local (recommended for now):
1. Install Ollama from ollama.com
2. `ollama pull llama3.2:1b`
3. `python server.py`

Option B, hosted model:

    set LLM_PROVIDER=anthropic        (Windows; use export on Mac/Linux)
    set ANTHROPIC_API_KEY=your-key
    python server.py

Other settings: `OLLAMA_MODEL` (for example `llama3.2:3b` for better answers), `OLLAMA_URL`, `ANTHROPIC_MODEL`, `LLM_PROVIDER=none` to use only scripted replies.
The assistant is told never to promise refunds, amounts or exact times, and customer text is treated as a message, not as instructions.

## Menu, search and delete
A menu on the left (a bar at the top on phones) lets you move around.
- Customer: Home, Report a problem, Recent reports, Talk to support.
- Support agent: Queue, Grouped issues, Recent reports, Resolved.
