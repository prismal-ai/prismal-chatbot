# prismal-chatbot — Architecture (Chatbot Bootstrap)

| Field | Value |
|---|---|
| **Status** | `DRAFT` |
| **Version** | 1.0 |
| **Date** | 2026-07-19 |
| **Companion** | [`PLAN.md`](./PLAN.md) · [`SPEC.md`](./SPEC.md) · [`TASKS.md`](./TASKS.md) |

---

## 1. Guiding principle — a face, not a brain

> **intelligence → engine (`prismal`); transport → `prismal-sdk`; serving the
> engine → `prismal-server`; bridging chat platforms to a conversation and
> rendering a reply → this repo. Nothing else lives here.**

`prismal-chatbot` talks to exactly **one** upstream engine surface — a
running `prismal-server`, and only **through `prismal_sdk`** — plus N
downstream chat platforms (Slack, Discord in v0.1) through their own
official SDKs. It never imports `prismal` (engine) or `prismal_server`
(host), never opens a raw HTTP connection to the host (no direct `httpx`
use against it), never parses SSE, never maps wire error codes, and never
builds a prompt. Its whole job is:

| Concern | Owner |
|---|---|
| Agent reasoning, RAG, tools, security | `prismal` (engine), server-side |
| Wire contract (REST/SSE, errors, auth headers) | `prismal-server` (SPEC-RHB-*) |
| Typed transport, SSE parsing, exceptions | `prismal-sdk` (SPEC-CSB-*) |
| Platform events, conversation↔thread mapping, throttled rendering | **this repo** (SPEC-CHB-*) |

Any pull toward importing `prismal.*` / `prismal_server.*`, calling the host
with `httpx` directly, re-implementing event parsing, or adding
prompt/agent/business logic is a design smell — see the `CLAUDE.md` review
checklist. This mirrors `prismal-webchat`'s ADR-002 one hop over: there the
BFF sits between a browser and the host; here the bridge sits between a
chat platform and the host.

---

## 2. C4 — Context

```
   Slack workspace                         Discord guild
  ┌──────────────────┐                   ┌──────────────────┐
  │ app_mention / DM │                   │ mention / DM      │
  └─────────┬─────────┘                   └─────────┬─────────┘
            │ Socket Mode (WSS)                      │ Gateway (WSS)
            ▼                                        ▼
  ┌────────────────────┐                   ┌────────────────────┐
  │  SlackAdapter        │                 │  DiscordAdapter      │
  └─────────┬───────────┘                   └─────────┬───────────┘
            │                                          │
            └──────────────┬───────────────────────────┘
                            ▼
                  ┌────────────────────────┐
                  │   ConversationBridge    │   this repo — process-wide,
                  │   (platform-agnostic)   │   prismal_chatbot core
                  └───────────┬─────────────┘
                              │ import prismal_sdk
                              │ (AsyncPrismalClient, HTTP + SSE)
                              ▼
                  ┌────────────────────────┐
                  │     prismal-server      │   sibling repo
                  └───────────┬─────────────┘
                              │ 4 seams
                              ▼
                  ┌────────────────────────┐
                  │     prismal (engine)    │
                  └────────────────────────┘
```

Both adapters run as independent asyncio tasks inside **one process** (v0.1
deploy unit). Neither platform ever talks to `prismal-server` directly —
all engine traffic originates from the bridge, which holds the service
credential(s). There is no browser and no CORS concern; the trust boundary
is each platform's own event-signing/Socket-Mode authentication.

---

## 3. C4 — Container / module layout

```
prismal-chatbot/
├── pyproject.toml                  # deps: prismal-sdk, slack-bolt, discord.py; NO prismal/prismal-server/httpx dep
├── CLAUDE.md                       # boundary rule + review checklist
├── README.md
├── Dockerfile                      # one-container deploy running both adapter tasks
├── prismal_chatbot/
│   ├── __init__.py
│   ├── __main__.py                 # entrypoint: builds bridge + enabled adapters, runs asyncio.gather
│   ├── settings.py                 # ChatbotSettings (pydantic-settings, PRISMAL_CHATBOT_ prefix; nested SlackSettings/DiscordSettings, SecretStr tokens)
│   ├── client.py                   # get_client(): lazily built, process-wide AsyncPrismalClient from settings
│   ├── bridge/
│   │   ├── conversation_bridge.py  # ConversationBridge: thread resolution, streaming turn, error mapping, timeout guard
│   │   ├── throttle.py             # StreamThrottle: coalesces TokenEvent deltas into a bounded edit cadence
│   │   └── models.py               # ConversationKey, BridgeError, TurnOutcome
│   ├── store/
│   │   └── thread_store.py         # ThreadStore (SQLite, WAL): (platform, external_conversation_key) -> thread_id
│   ├── platforms/
│   │   ├── base.py                 # PlatformAdapter Protocol, Responder Protocol (send_placeholder/update/finalize/error)
│   │   ├── slack_adapter.py        # SlackAdapter (slack_bolt, Socket Mode): events -> bridge.handle_turn()
│   │   └── discord_adapter.py      # DiscordAdapter (discord.py, Gateway): events -> bridge.handle_turn()
│   └── errors.py                   # SDK exception -> friendly-text mapping (shared by all platforms)
├── docs/
│   ├── configuration.md            # PRISMAL_CHATBOT_* reference (server + per-platform)
│   ├── slack-setup.md              # Slack app manifest, scopes, Socket Mode token
│   ├── discord-setup.md            # Discord application, intents, bot token
│   └── deploy.md                   # Docker, restart policy, ThreadStore volume
└── tests/
    ├── conftest.py                 # FakeAsyncPrismalClient, FakeSlackClient, FakeDiscordClient, tmp ThreadStore
    ├── unit/                       # bridge, throttle, thread store, settings, error mapping (offline)
    └── contract/                   # opt-in: real prismal-server round-trip via the bridge
```

**Why a bridge and not per-platform business logic:** the entire
platform-independent part of a turn — resolve or create a `thread_id`,
stream `send_message()`, coalesce tokens, map errors, enforce the timeout —
is identical for Slack and Discord. `ConversationBridge` owns it once;
each `PlatformAdapter` only translates platform events into a
`ConversationKey` + text, and platform responder calls
(`send_placeholder`/`update`/`finalize`/`error`) into platform API calls.
Adding a third platform is: implement `PlatformAdapter` + `Responder`, wire
it in `__main__.py`. No change to the bridge.

---

## 4. Data flow

### 4.1 Streaming conversation turn (RF-CHB-001/002/003/005)

```
platform event (mention/DM) arrives at SlackAdapter / DiscordAdapter
  │ adapter builds ConversationKey(platform, external_conversation_key)
  │ adapter calls responder.send_placeholder() -> platform message id
  └─ await bridge.handle_turn(key, text, responder):
        thread_id = await store.get(key)
        if thread_id is None:
            thread = await client.create_thread()
            await store.put(key, thread.thread_id)      # persist BEFORE first send
            thread_id = thread.thread_id
        throttle = StreamThrottle(responder.update, interval_s=…, min_chars=…)
        async for event in client.send_message(thread_id, text):
            with timeout guard (PRISMAL_CHATBOT_TURN_TIMEOUT_S):
                TokenEvent(delta)        → throttle.feed(delta)              (coalesced edit)
                ToolCallEvent(name,args) → responder.update(f"🔧 using {name}…")  (transient, best-effort)
                StateEvent(data)         → ignored in v0.1 (logged debug)
                DoneEvent(finish_reason) → throttle.flush(); responder.finalize(final_text)
        except PrismalSDKError as e:
            → responder.error(friendly_text(e))          (errors.py mapping)
        except TimeoutError:
            → responder.error("This is taking longer than expected — try again.")
```

`store.put()` happens **before** the first token is requested, so a crash
mid-turn still leaves the mapping durable for the retry/next message on the
same conversation — the engine's checkpointer is the source of truth for
message history, `ThreadStore` only remembers *which* thread a platform
conversation maps to.

### 4.2 Streaming throttle (RF-CHB-005)

```
StreamThrottle.feed(delta):
    buffer += delta
    if (now - last_flush) >= interval_s  OR  len(buffer) >= min_chars:
        responder.update(accumulated_text)   # one platform edit call
        last_flush = now
StreamThrottle.flush():
    if buffer not yet rendered: responder.update(accumulated_text)  # final safety flush before finalize()
```

Interval/char thresholds are per-platform config
(`PRISMAL_CHATBOT_SLACK_EDIT_INTERVAL_S`,
`PRISMAL_CHATBOT_DISCORD_EDIT_INTERVAL_S`, §6 `SPEC.md`), defaulting to a
value comfortably under each platform's documented edit-rate budget. This
is the chatbot's analogue of the webchat's "one `yield` per event" — but
coalesced, because a chat platform message is not a WebSocket state patch,
it is a rate-limited API call.

### 4.3 Cancellation / turn bound (RF-CHB-008)

```
No platform gives the bridge a "stop" affordance like the webchat's button.
Instead: PRISMAL_CHATBOT_TURN_TIMEOUT_S bounds every turn. On expiry the
async-for is cancelled (asyncio.timeout / wait_for) → the SDK closes the
HTTP stream (SPEC-CSB-CHT-006) → the host cancels the run
(SPEC-RHB-CHT-004) → the same chain the webchat relies on, triggered by a
deadline instead of a user click.
```

### 4.4 Conversation identity (RF-CHB-004)

```
ConversationKey = (platform: "slack"|"discord", external_conversation_key: str)
  Slack:   external_conversation_key = f"{team_id}:{channel_id}:{thread_ts or ts}"
  Discord: external_conversation_key = f"{guild_id or 'dm'}:{channel_id}"

ThreadStore (SQLite, WAL mode):
  CREATE TABLE thread_map (
    platform TEXT NOT NULL,
    external_conversation_key TEXT NOT NULL,
    thread_id TEXT NOT NULL,
    created_at TEXT NOT NULL,
    PRIMARY KEY (platform, external_conversation_key)
  );
```

One row per external conversation, for the lifetime of the deployment.
Deleting a row (manual op, not exposed as a bot command in v0.1) starts a
fresh engine conversation on the next message — the reset path is
documented in `docs/deploy.md`, not built as a chat command (out of scope,
see PLAN §5).

### 4.5 Configuration & secret path (RF-CHB-007)

```
env (PRISMAL_CHATBOT_*) ──▶ ChatbotSettings (pydantic-settings)
  ├─ server_url, token (SecretStr), org_id ──▶ client.py → AsyncPrismalClient
  ├─ slack.bot_token / slack.app_token (SecretStr), slack.enabled ──▶ SlackAdapter
  ├─ discord.bot_token (SecretStr), discord.enabled ──▶ DiscordAdapter
  └─ turn_timeout_s, edit_interval_s (per platform) ──▶ ConversationBridge / StreamThrottle
```

Every credential is a `SecretStr`. Adapters read tokens once at startup to
open their platform connection; no credential is ever echoed into a chat
message, log line, or exception text (enforced by the secret-leak test,
mirroring `prismal-webchat`'s `SPEC-WCB-AUT-002`).

---

## 5. Architecture Decision Records

### ADR-001 — A platform-agnostic `ConversationBridge`, adapters stay thin
**Decision:** All turn logic (thread resolution/persistence, streaming
consumption, throttling, error/timeout mapping) lives in one
`ConversationBridge` shared by every platform. `SlackAdapter` /
`DiscordAdapter` implement only event intake (`ConversationKey` + text
extraction) and a `Responder` (placeholder/update/finalize/error) backed by
that platform's API.
**Rationale:** the turn lifecycle against `prismal-sdk` is identical
regardless of platform (same pattern as `prismal-webchat`'s `ChatState`);
duplicating it per adapter would drift. Isolating platform SDKs behind a
`Responder` Protocol keeps `slack_bolt`/`discord.py` imports out of the
bridge and out of its unit tests.
**Alternatives:** one monolithic Slack-only bot with Discord bolted on
later (rejected — the presentation scopes both platforms for v0.1; a bridge
seam costs little and pays off immediately). **Status:** Accepted.

### ADR-002 — Durable `ThreadStore` (SQLite) keyed by external conversation
**Decision:** Persist `(platform, external_conversation_key) -> thread_id`
in a local SQLite file, written before the first `send_message()` call for
a new conversation.
**Rationale:** unlike a browser tab (`prismal-webchat` ADR-004), a Slack
channel or Discord DM outlives the bot process; without persistence a
redeploy silently forks every ongoing conversation. SQLite is the smallest
dependency that gives crash-safe durability for a single-process deploy.
**Alternatives:** in-memory dict (loses history on every restart — rejected
as a correctness bug, not just a UX one); external DB / Redis (real
infrastructure with no current multi-replica need — deferred to
CHB-FUTURE-06 if load demands it). **Status:** Accepted.

### ADR-003 — Coalesced streaming instead of per-token edits
**Decision:** `StreamThrottle` buffers `TokenEvent` deltas and flushes to
the platform on a bounded interval/size cadence, not on every token; the
final `DoneEvent` always forces one last flush.
**Rationale:** Slack and Discord both rate-limit message edits far below
per-token granularity; naive per-token edits would 429 within seconds on
any non-trivial answer. This is the direct platform-imposed counterpart to
the webchat's "one `yield` per event" (ADR-omitted there because a
WebSocket state patch has no comparable external rate limit).
**Alternatives:** post a new message per chunk instead of editing (spams
the channel — rejected); no throttling (breaks in production — rejected).
**Status:** Accepted.

### ADR-004 — One process, two adapter tasks (v0.1 deploy unit)
**Decision:** `__main__.py` starts `SlackAdapter` and `DiscordAdapter` (each
independently enabled/disabled by settings) as sibling `asyncio` tasks in
one process; one Dockerfile, one container.
**Rationale:** v0.1 targets one workspace and one guild — running two
processes for that is unnecessary operational overhead; asyncio tasks share
the process's `AsyncPrismalClient` and `ThreadStore` cleanly. **Alternatives:**
separate deployable per platform (revisit if/when multi-workspace/
multi-guild scale, CHB-FUTURE-01/06, makes a shared process a bottleneck).
**Status:** Accepted.

### ADR-005 — Turn timeout instead of a stop affordance
**Decision:** every turn is bounded by `PRISMAL_CHATBOT_TURN_TIMEOUT_S`;
there is no explicit "stop" command in v0.1.
**Rationale:** chat platforms have no first-class equivalent to a button
click mid-stream; a deadline gives the same "don't hang forever" guarantee
the webchat's stop button gives, using the identical cancellation chain
(early stream closure → SDK closes HTTP → host cancels the run).
**Alternatives:** a `/stop` slash command (real feature, deferred to
CHB-FUTURE-03 with the rest of interactive components). **Status:** Accepted.

### ADR-006 — Fake platform clients as the unit-test seam
**Decision:** Unit tests inject a `FakeAsyncPrismalClient` (as in
`prismal-webchat`) **and** fake Slack/Discord clients that record
placeholder/update/finalize/error calls without opening a socket; no live
platform connection and no network in the unit suite.
**Rationale:** mirrors the engine's factory-injection testing pattern and
the webchat's `FakeAsyncPrismalClient` (`ADR-005` there); the bridge's
testable surface is exactly "platform event + SDK events in → responder
calls + `ThreadStore` state out". **Status:** Accepted.

---

## 6. Error handling

- Every `PrismalSDKError` raised during a turn is caught in exactly one
  place — `ConversationBridge.handle_turn()` — and mapped
  (`errors.py`) to a platform-agnostic friendly string, which each
  `Responder.error()` posts however is idiomatic on that platform (a reply
  message on both Slack and Discord in v0.1).
- Mapping mirrors `prismal-webchat`'s (`ARCHITECTURE.md` §6):
  `PrismalConnectionError` / `PrismalTimeoutError` → "Can't reach the
  assistant right now — try again."; `BudgetExceededError` → "The assistant
  is over capacity for now."; `PolicyDeniedError` / `PermissionDeniedError`
  → "That request can't be processed."; any other `PrismalAPIError` /
  `PrismalSDKError` → generic failure text. Codes, stack traces, and server
  messages go to the bot's logs only.
- A mid-stream error preserves whatever partial text was already flushed to
  the platform message (the placeholder is edited to show the partial
  answer plus an appended "⚠️ …" note) rather than replacing it — matching
  the webchat's "never blank a partial answer" rule.
- There is no retry affordance beyond "ask again" — chat platforms have no
  standard inline-retry UI; this is an intentional, documented UX
  difference from the webchat (which has a retry button).

---

## 7. Observability

Structured logs (turn started/finished, platform, `thread_id`, event
counts, mapped error codes, edit count per turn — never message content,
never any token) via standard `logging`, mirroring `prismal-webchat`'s
approach. The bot does not stand up its own OTel stack in v0.1; engine-side
tracing (Langfuse/OTel) already covers the interesting part of every turn.
A per-turn request id correlates bot logs with host logs. Revisit if the
bot grows fleet concerns (CHB-FUTURE-06).

---

## 8. Packaging & deploy (sketch)

`uv`-based project; `python -m prismal_chatbot` runs both adapter tasks for
dev, Dockerfile for prod (single container, `ThreadStore` SQLite file on a
mounted volume so restarts do not lose conversation identity — see
`docs/deploy.md`). Not published to PyPI — the chatbot is a deployable
application, not a library, matching `prismal-webchat`. Versioning follows
the org's `MAJOR.MINOR.PATCH`; `v0.1.0` targets the M5 exit criterion in
`PLAN.md`. Pins: `prismal-sdk>=0.1,<1`, `slack-bolt`, `discord.py` (all
bumped deliberately).

---

## Change History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-07-19 | Ernesto Crespo | Initial architecture: platform-agnostic bridge over prismal-sdk, Slack (Socket Mode) + Discord (Gateway) adapters, durable ThreadStore, coalesced streaming throttle |
