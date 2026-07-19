# prismal-chatbot

External chat-platform bots for the
[prismal](https://github.com/prismal-ai/prismal) ecosystem — a long-running
process that bridges **Slack** and **Discord** to a running
[`prismal-server`](https://github.com/prismal-ai/prismal-server), exclusively
through [`prismal-sdk`](https://github.com/prismal-ai/prismal-sdk)
(`AsyncPrismalClient`).

There is no browser here: each platform's own real-time transport (Slack
Socket Mode, Discord Gateway) delivers events to a platform adapter, which
hands them to a shared, platform-agnostic `ConversationBridge`. The bridge
resolves or creates one engine `thread_id` per external conversation,
persists that mapping in a local SQLite `ThreadStore` (so a restart never
forks an ongoing conversation), streams the reply through `prismal-sdk`, and
renders it back with platform-rate-limit-safe throttled message edits.

> **Status: bootstrap (SDD seed).** The design is complete in
> [`specs/chatbot-bootstrap/`](./specs/chatbot-bootstrap/)
> (`PLAN.md` · `ARCHITECTURE.md` · `SPEC.md` · `TASKS.md`); implementation
> phases CHB-00…04 are tracked in `TASKS.md`.

## Ecosystem position

```
Slack workspace ── Socket Mode ──▶ SlackAdapter   ─┐
Discord guild   ── Gateway ──────▶ DiscordAdapter ─┼──▶ ConversationBridge (this repo)
                                                     │      prismal_sdk (HTTP + SSE)
                                                     ▼
                                              prismal-server ──▶ prismal (engine)
```

| Repo | Role | Status |
|---|---|---|
| [`prismal`](https://github.com/prismal-ai/prismal) | Engine (LangGraph supervisor, 29 routes, RAG, security) | v3.10.1 — 21/21 specs |
| [`prismal-server`](https://github.com/prismal-ai/prismal-server) | FastAPI host (REST + SSE + A2A) | v0.1 complete |
| [`prismal-sdk`](https://github.com/prismal-ai/prismal-sdk) | Typed Python client (sync + async) | bootstrap complete |
| [`prismal-webchat`](https://github.com/prismal-ai/prismal-webchat) | Embeddable browser chat widget | SDD seed |
| `prismal-chatbot` | External bots — Slack, Discord (this repo) | **SDD seed** |

## What v0.1 delivers

A Slack adapter (Socket Mode — no public inbound endpoint needed) and a
Discord adapter (Gateway), both running as asyncio tasks in one process.
Mention the bot or DM it and get a streamed, throttled, edited reply in
Slack threads or Discord messages. Conversation identity survives a
restart. Friendly errors on failure, a per-turn timeout instead of a stop
button (chat platforms have no equivalent affordance). See
`specs/chatbot-bootstrap/PLAN.md` §5 for the full in/out-of-scope list.

## Quickstart (dev)

Requires Python 3.13 (dev target), [`uv`](https://docs.astral.sh/uv/), a
running `prismal-server` (defaults to `http://localhost:8000`), and a Slack
app and/or Discord application to test against.

```bash
uv pip install -e ".[dev]"

export PRISMAL_CHATBOT_SERVER_URL="http://localhost:8000"
export PRISMAL_CHATBOT_TOKEN="…"                # service credential

export PRISMAL_CHATBOT_SLACK_ENABLED=true
export PRISMAL_CHATBOT_SLACK_BOT_TOKEN="xoxb-…"
export PRISMAL_CHATBOT_SLACK_APP_TOKEN="xapp-…" # Socket Mode

export PRISMAL_CHATBOT_DISCORD_ENABLED=true
export PRISMAL_CHATBOT_DISCORD_BOT_TOKEN="…"

python -m prismal_chatbot
```

Each adapter is independently enabled/disabled — set only the platform(s)
you need. See `docs/slack-setup.md` and `docs/discord-setup.md` (Phase 4)
for app manifest / scopes / intents.

## Configuration

| Env var | Default | Meaning |
|---|---|---|
| `PRISMAL_CHATBOT_SERVER_URL` | `http://localhost:8000` | Target `prismal-server` |
| `PRISMAL_CHATBOT_TOKEN` | — | Service bearer credential (never posted to a platform) |
| `PRISMAL_CHATBOT_ORG_ID` | — | Tenant for all bot traffic (`X-Org-Id`) |
| `PRISMAL_CHATBOT_TURN_TIMEOUT_S` | `120` | Per-turn bound (no stop button on chat platforms) |
| `PRISMAL_CHATBOT_STORE_PATH` | `./data/thread_store.db` | SQLite conversation↔thread mapping |
| `PRISMAL_CHATBOT_SLACK_ENABLED` | `false` | Enable the Slack adapter |
| `PRISMAL_CHATBOT_SLACK_BOT_TOKEN` | — | Slack bot token (`xoxb-…`) |
| `PRISMAL_CHATBOT_SLACK_APP_TOKEN` | — | Slack app-level token for Socket Mode (`xapp-…`) |
| `PRISMAL_CHATBOT_SLACK_EDIT_INTERVAL_S` | `1.0` | Streaming-edit throttle |
| `PRISMAL_CHATBOT_DISCORD_ENABLED` | `false` | Enable the Discord adapter |
| `PRISMAL_CHATBOT_DISCORD_BOT_TOKEN` | — | Discord bot token |
| `PRISMAL_CHATBOT_DISCORD_EDIT_INTERVAL_S` | `1.0` | Streaming-edit throttle |

Full reference: `specs/chatbot-bootstrap/SPEC.md` §8.

## Development

```bash
uv run pytest                       # unit suite — offline, fake SDK + fake platform clients
uv run pytest -m contract           # opt-in — bridge against a running prismal-server
uv run ruff check . && uv run mypy prismal_chatbot
```

Boundary rules (no `prismal`/`prismal_server` imports, no direct HTTP to the
host, platform SDKs isolated in `platforms/`, no secret ever logged or
posted) are enforced by tests — see [`CLAUDE.md`](./CLAUDE.md) for the
review checklist.

## License

[MIT](./LICENSE)
