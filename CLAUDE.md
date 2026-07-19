# CLAUDE.md — prismal-chatbot

Guidance for Claude Code (claude.ai/code) working in this repository.

## What this repo is

`prismal-chatbot` is the **external bots** front-end for the `prismal`
ecosystem: a long-running Python process that bridges chat platforms —
**Slack** and **Discord** in v0.1 — to a running
[`prismal-server`](https://github.com/prismal-ai/prismal-server), exclusively
through [`prismal-sdk`](https://github.com/prismal-ai/prismal-sdk)
(`AsyncPrismalClient`). It is the sibling of `prismal-webchat` (browser BFF)
one layer further out: same upstream contract, different downstream surface.

The full design lives in
[`specs/chatbot-bootstrap/`](./specs/chatbot-bootstrap/) (`PLAN.md` ·
`ARCHITECTURE.md` · `SPEC.md` · `TASKS.md`). Read those before implementing.

## The one hard rule — a face, not a brain

> **intelligence → engine (`prismal`); transport → `prismal-sdk`; serving →
> `prismal-server`; bridging chat platforms to a conversation and rendering
> a reply → this repo. Nothing else lives here.**

This repo imports **nothing** from `prismal` (engine) or `prismal_server`
(host), and never opens its own HTTP connection to the host (no direct
`httpx` against it). All upstream traffic goes through `prismal_sdk`'s typed
client. Platform SDKs (`slack_bolt`, `discord.py`) are isolated inside
`platforms/*_adapter.py` — the bridge and store never import them. Adding
agent/RAG/tool logic, prompt construction, SSE parsing, wire error-code
mapping, or chatbot-side conversation *content* persistence (only the
conversation↔`thread_id` mapping is persisted, never messages) is a bug.

| Concern | Owner |
|---|---|
| Agent reasoning, RAG, tools, security layers | `prismal` (engine, server-side) |
| Wire contract (REST/SSE, errors, auth) | `prismal-server` (`SPEC-RHB-*`) |
| Typed transport, SSE parsing, exceptions | `prismal-sdk` (`SPEC-CSB-*`) |
| Platform events, conversation↔thread mapping, throttled rendering | **this repo** (`SPEC-CHB-*`) |

## Review checklist (enforce on every PR)

- [ ] No import of `prismal` or `prismal_server`, and no direct `httpx` use
      against `prismal-server`, anywhere in `prismal_chatbot/` — the AST
      boundary-guard test must stay green (`SPEC-CHB-APP-002`).
- [ ] No platform SDK (`slack_bolt`, `discord.py`) imported outside
      `platforms/*_adapter.py`; `bridge/` and `store/` stay platform-agnostic
      (`SPEC-CHB-APP-005`).
- [ ] User-provided text reaches `send_message(content=…)` **unmodified** —
      no templating, prefixing, or prompt building anywhere (`SPEC-CHB-BRG-005`).
- [ ] `ThreadStore.put()` happens **before** the first `send_message()` call
      for a new conversation — never after, or a crash mid-turn loses the
      mapping (`SPEC-CHB-STO-001/003`, `ADR-002`).
- [ ] Streaming replies go through `StreamThrottle`, never a raw per-token
      platform edit — naive per-token edits will hit Slack/Discord rate
      limits in production (`SPEC-CHB-THR-*`, `ADR-003`).
- [ ] Every turn is bounded by `PRISMAL_CHATBOT_TURN_TIMEOUT_S`; a stream
      that never reaches `DoneEvent` must still terminate and report a
      friendly error (`SPEC-CHB-BRG-004`).
- [ ] No secret — `PRISMAL_CHATBOT_TOKEN`, Slack bot/app tokens, Discord bot
      token — ever appears in a log line, error message, or a message
      posted to a platform (`SecretStr` everywhere; `SPEC-CHB-AUT-002`).
- [ ] SDK exceptions are caught only in `ConversationBridge.handle_turn()`
      and rendered as friendly text via `errors.py`; no codes, server
      messages, or tracebacks reach a channel/DM (`SPEC-CHB-ERR-*`).
- [ ] A mid-stream error preserves whatever partial text was already
      flushed — never blank or delete a partially-rendered answer
      (`SPEC-CHB-ERR-003`).
- [ ] One adapter task crashing must not take down the other adapter or the
      process — isolated task supervision (`SPEC-CHB-NFR-004`).
- [ ] Fully async; no blocking calls in adapter event handlers or the
      bridge's streaming loop.
- [ ] New behaviour is **test-first (TDD)**: failing test → minimal code →
      green. Unit tests use `FakeAsyncPrismalClient` + fake Slack/Discord
      clients (offline, no sockets); anything touching a live server or a
      live platform connection goes in the opt-in `tests/contract/` /
      manual-verification path (`SPEC-CHB-NFR-001`).
- [ ] Dep pins stay `prismal-sdk>=0.1,<1` plus a tested `slack-bolt` /
      `discord.py` minor; bumps are deliberate PRs (`SPEC-CHB-NFR-005`).
- [ ] `ruff check .` and `mypy prismal_chatbot` are clean.
- [ ] New capability traces to a `SPEC-CHB-*` requirement — if the wire
      contract must change, that is negotiated in `prismal-server` /
      `prismal-sdk` first, never invented here.

## Common commands

```bash
uv pip install -e ".[dev]"
python -m prismal_chatbot            # runs enabled adapter(s) as asyncio tasks
uv run pytest                        # unit (offline, fake SDK + fake platform clients)
uv run pytest -m contract            # opt-in: bridge against a running prismal-server
uv run ruff check . && uv run ruff format --check .
uv run mypy prismal_chatbot
```

Local end-to-end: start a `prismal-server`
(`uvicorn prismal_server.app:app`), point this repo at it via
`PRISMAL_CHATBOT_SERVER_URL`, enable the platform(s) you have credentials
for, and mention the bot in a real Slack workspace / Discord guild.

## Boundary examples

- Need the answer streamed to the channel? **Already have it** — iterate
  `client.send_message(...)` inside `ConversationBridge.handle_turn()` and
  feed deltas to `StreamThrottle`; never parse SSE yourself, never post a
  new message per token.
- Need conversation history after a restart? **No chatbot-side history DB**
  — message history lives in the engine's checkpointer under the resolved
  `thread_id`; this repo only persists *which* `thread_id` a Slack
  channel/Discord DM maps to (`ThreadStore`), never the messages themselves.
- Want to improve answers with a system prompt or RAG tweak? **No** — that
  is engine configuration, server-side. This repo passes user text through
  unmodified.
- Need a `/stop` command or interactive buttons? **Not in v0.1** — chat
  platforms have no built-in mid-stream cancel affordance; v0.1 uses a
  bounded per-turn timeout instead. Slash commands / interactive components
  are `CHB-FUTURE-03`.
- Want to add Telegram or WhatsApp? **Additive, not a rewrite** — implement
  `PlatformAdapter` + `Responder` (`platforms/base.py`) and wire it in
  `__main__.py`; the bridge, throttle, and `ThreadStore` are already
  platform-agnostic (`CHB-FUTURE-02`).
- Need to serve more than one Slack workspace or Discord guild? **Not in
  v0.1** — one static bot token per platform per deployment; multi-tenant
  OAuth install is `CHB-FUTURE-01`.
