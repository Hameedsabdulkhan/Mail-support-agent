# AI Customer Support

An AI-powered customer support backend for a fictional e-commerce company (Harbor & Pine, a home and outdoor seller on Amazon). It watches a Gmail inbox, reads each new customer email, looks up answers in a knowledge base, and either replies or hands the email to a human.

> **Principle: the model proposes, code decides.** The LLM drafts and classifies. Plain Python applies the rules and makes the final call on whether to reply or escalate.



At the moment `app/pipeline.py` is a stub: it logs the sender and subject of each new email and adds a "processed" label. It does not send replies yet.

## How it works

```
Gmail inbox
    │  (polled every N seconds)
    ▼
Poller ──> Guards ──> Classifier (small model) ──> Agent (larger model + KB tools)
                          │                              │
                          │ needs human?                 │ drafts structured reply
                          ▼                              ▼
                     Escalate  <────────  Python rules decide  ────────>  Reply
                  (acknowledge + label)                                (sent in same thread)
```

1. **Poller** asks Gmail for new mail on a fixed interval and labels what it has handled.
2. **Guards** (plain Python) stop the bot from answering itself or other automated mail.
3. **Classifier** (small, fast model) sorts the email into a section such as paddle boards, shelving, returns, or shipping, and checks the always-escalate rules (angry customers, refund requests, safety issues, prompt injection, and so on).
4. **Agent** (stronger model) reads the knowledge base through tools and drafts a reply as structured output.
5. **Python decides.** Replies that promise an action (refund, shipment, replacement) or that the knowledge base cannot answer are escalated to a person.
6. Every email and event is stored in the database.

Escalated threads get a `needs-human` label and a short acknowledgement for the customer. A colleague replies from Gmail and removes the label, and the AI takes the thread back.

The server only makes outgoing calls. It needs no webhooks, no domain, and no open web port.

## Tech stack

- Python and [uv](https://docs.astral.sh/uv/)
- FastAPI
- Gmail API (OAuth 2.0)
- [Pydantic AI](https://ai.pydantic.dev/) with Anthropic models
- Supabase (Postgres)
- Docker Compose on a VPS
- React admin dashboard

## Prerequisites

- Python 3.13 (the guide pins this) and uv
- Node 22 or later (for the dashboard)
- Git
- A **new, separate Gmail account** for the demo. Do not use your personal inbox.
- A Supabase account (the free plan is enough)
- An Anthropic API key

## Setup

### 1. Install dependencies

```bash
git clone <your-repo-url>
cd ai-customer-support
uv sync
```

### 2. Get Gmail API access

1. Create a Google Cloud project and enable the **Gmail API**.
2. Set up the OAuth consent screen (**External**, left in **Testing**) and add the demo Gmail address as a test user.
3. Add the scope `https://www.googleapis.com/auth/gmail.modify`.
4. Create a **Desktop app** OAuth client, download the JSON, and save it as `credentials.json` in the project root.
5. Run the one-time login script and sign in with the demo account:

   ```bash
   uv run python scripts/get_refresh_token.py
   ```

6. Copy the three printed values into `.env`.

### 3. Configure the environment

Copy `.env.example` to `.env` and fill it in:

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REFRESH_TOKEN=
DATABASE_URL=
ANTHROPIC_API_KEY=
CLASSIFIER_MODEL=anthropic:claude-haiku-5-5
AGENT_MODEL=anthropic:claude-sonnet-5-5
POLL_INTERVAL_SECONDS=30
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=
PYDANTIC_AI_NO_BANNER=1
```

Models are plain settings, so swapping one is a change in `.env`, not in code. Use a short poll interval (30 seconds) while developing. In real use, 600 seconds is plenty.

## Running

Run the poller:

```bash
uv run python -m app.poller
```

Run it with auto-restart whenever you edit code in `app/` or change `.env`:

```bash
uv run watchfiles "uv run python -m app.poller" app .env
```

Run the API (health check at `http://localhost:8000/health`):

```bash
uv run fastapi dev app/main.py
```

On startup the poller logs `Polling every N seconds`, then `Handled N new messages` when mail arrives.

## Testing

**Gmail check script.** Connects to the inbox, reads the newest unprocessed email, and sends a reply. It sends a real email, so run it only against a test message you sent from another account of your own.

**Playground (once the AI pipeline is built).** Send a question straight to the decision function without Gmail:

```bash
uv run python playground/try_pipeline.py
```

**Evals (once built).** A set of golden emails checks every pipeline decision, so you can change a prompt or swap a model and see whether quality drops.

## Project structure

```
ai-customer-support/
├── app/
│   ├── ai/              # classifier, agent, rules, decision logic (planned)
│   ├── __config__.py    # settings loaded from .env
│   ├── gmail.py         # Gmail client
│   ├── guards.py        # skip rules (planned)
│   ├── pipeline.py      # per-email handling
│   ├── poller.py        # polling loop
│   └── main.py          # FastAPI app
├── scripts/
│   ├── get_refresh_token.py   # one-time Gmail login
│   └── gmail_check.py         # manual Gmail test
├── playground/          # try the pipeline without email (planned)
├── .env                 # local secrets (not committed)
├── .env.example
└── pyproject.toml
```

## Security

- `.env`, `credentials.json`, and any token files must stay out of git. Check `.gitignore`.
- Never share screenshots of your terminal after running `get_refresh_token.py`, because it prints secrets.
- On the deployed server, nothing should be reachable from the internet except SSH from your own IP address.

## Troubleshooting

**`ModuleNotFoundError: No module named 'google.oauth2'`**
The packages went into the wrong Python. Install with `uv add google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client` (not plain `pip`) and start the app through `uv run`.

**`invalid_grant` error from the poller**
In Google's Testing mode, refresh tokens expire after 7 days. Run `scripts/get_refresh_token.py` again and update `GOOGLE_REFRESH_TOKEN` in `.env` (on the server too).

**Printed refresh token is `None`, or `403: access_denied`**
Check that the demo address is listed under **Test users** in the Google Auth platform audience settings, and that the login script uses `prompt="consent"`.

