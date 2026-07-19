# prismal-chatbot — External Bots (Chatbot Bootstrap)

## Strategic Plan / Product Requirements Document (PLAN)

| Field | Value |
|---|---|
| **Author** | Ernesto Crespo |
| **Status** | `DRAFT` |
| **Version** | 1.0 |
| **Date** | 2026-07-19 |
| **Phase** | CHB (Chatbot Bootstrap) |
| **Repository** | `prismal-ai/prismal-chatbot` (this repo) |
| **Upstream contract** | `prismal-sdk >= 0.1, < 1` (typed client over the wire contract fixed by `prismal-server/specs/reference-host-bootstrap/SPEC.md`) |
| **Reviewers** | Tech Lead, AI Architect |
| **Priority** | P2 (second end-user-facing front-end; ships after `prismal-webchat`) |
| **Related** | `prismal-ai/prismal` (engine, v3.10.1, 21/21 specs); `prismal-ai/prismal-server` (host, v0.1 complete); `prismal-ai/prismal-sdk` (client, bootstrap complete); `prismal-ai/prismal-webchat` (sibling front-end, SDD seed — shares the BFF/"face not a brain" pattern); siblings `prismal-tui`, `prismal-dashboard` (`PENDIENTE`) |

---

## 1. Executive Summary

`prismal` (the engine) is a pure LangGraph library. `prismal-server` put it on
the network with a fixed wire contract (REST + SSE chat, standard A2A v0.3.x).
`prismal-sdk` wrapped that contract in a typed Python client (`PrismalClient` /
`AsyncPrismalClient`) so that every front-end repo talks to the engine the
same narrow way. `prismal-webchat` proved the pattern for a browser-embedded
widget.

`prismal-chatbot` is the ecosystem's **external-bot front-end**: a long-running
Python process that bridges chat platforms — **Slack** and **Discord** in v0.1
— to a running `prismal-server`, exclusively through `prismal-sdk`. Unlike the
webchat, there is no browser and no per-tab session: the bot listens for
platform events (mentions, DMs) over each platform's own real-time transport
(Slack Socket Mode, Discord Gateway), maps each external conversation to one
engine `thread_id`, streams the reply back with platform-appropriate
throttled message edits, and persists the conversation↔`thread_id` mapping
so a bot restart does not fork a new conversation.

This PLAN scopes **v0.1**: a Slack adapter and a Discord adapter running as
two asyncio tasks inside one process (single deployable), a shared
platform-agnostic bridge that owns the `prismal-sdk` turn lifecycle, a
local `ThreadStore` (SQLite) for conversation↔thread persistence, and
config/secrets per platform. `ARCHITECTURE.md`, `SPEC.md`, and `TASKS.md` in
this folder refine it into a buildable, test-first plan.

---

## 2. Context and Problem

- Per the `prismal-ai` ecosystem presentation (2026-07-07, v3.10.1): 8
  repositories. `prismal` and `prisma-notebooks` are `LISTO`; `prismal-server`
  v0.1 is complete; `prismal-sdk` bootstrap is complete. `prismal-webchat`,
  `prismal-tui`, `prismal-dashboard`, and `prismal-chatbot` are `PENDIENTE`.
  The ecosystem diagram places `prismal-chatbot` at the same layer as
  `prismal-tui`, `prismal-webchat`, and `prismal-dashboard` — all four consume
  `prismal-sdk`, never the raw wire contract or the engine directly. The
  presentation labels `prismal-chatbot` explicitly: **"External bots
  (Slack/Discord)."**
- `prismal-webchat`'s SDD already established the reusable shape for a
  front-end in this ecosystem: a thin BFF that owns *no* agent/business
  logic, talks to `prismal-server` only via `prismal_sdk`, and translates the
  SDK's typed streaming events into a surface-specific rendering. This repo
  reuses that shape but the surface is a chat **platform API**, not a
  browser: no HTML/CSS/embedding, no `rx.State`, but a comparable "one
  external conversation → one `thread_id`" mapping and a comparable
  streaming-turn → typed-events → UI-update pipeline (§4.2, `ARCHITECTURE.md`).
- Two problems the webchat's design does not solve, because a browser tab
  does not have them:
  1. **Conversation identity outlives the process.** A browser tab's
     `thread_id` only has to survive the tab; a Slack channel or Discord
     server exists indefinitely and the bot process restarts (deploys,
     crashes). Without persistence, every restart would silently start a new
     engine conversation for an ongoing Slack thread.
  2. **Streaming is not free-form.** Slack and Discord rate-limit message
     edits (roughly ~1/sec per message on Slack; Discord enforces per-route
     rate limits too) — token-by-token edits like the webchat's live SSE
     render would get throttled or banned. The bridge must coalesce token
     deltas into a bounded edit cadence.
- Without `prismal-chatbot`, the only ways to reach a running `prismal-server`
  from a chat platform are ad hoc scripts or a generic A2A peer — nothing a
  team can install into a real Slack workspace or Discord server today.

---

## 3. Target Users

- **Teams running Slack or Discord** who want a `prismal`-backed assistant
  reachable by mentioning it in a channel or DMing it, without touching the
  engine's wire contract.
- **The prismal-ai org itself** — a second reference front-end proving the
  `prismal-sdk` contract generalizes beyond a browser BFF.
- **`prismal-dashboard` / `prismal-tui` maintainers** — the conversation↔
  `thread_id` persistence pattern and the streaming-throttle pattern
  established here are reusable wherever a front-end outlives a single
  session.

---

## 4. Goals and Success Metrics

| Goal | Metric | Target |
|---|---|---|
| Reach the engine from Slack | Mentioning the bot (or DMing it) in a workspace produces a streamed, edited reply | End-to-end against a local `prismal-server`, no manual polling |
| Reach the engine from Discord | Mentioning the bot (or DMing it) in a guild produces a streamed, edited reply | Same, against Discord Gateway |
| Durable conversation identity | A bot restart mid-conversation continues the same engine thread | `ThreadStore` round-trip test: same external key → same `thread_id` after process restart |
| Rate-limit-safe streaming | Message edits stay under each platform's edit-rate budget | Throttle unit test asserts a bounded edit count for a scripted long token stream |
| Zero wire-contract code | No SSE parsing, error-code mapping, or JSON-RPC framing in this repo | All transport goes through `prismal_sdk`; enforced by an AST import/usage-guard test |
| Offline-testable | Unit suite runs with no live server and no live platform connection | Fake `AsyncPrismalClient` + fake platform clients; opt-in `contract` suite hits a real host |
| Installable via uv | `uv pip install -e ".[dev]"` + one entrypoint boots both adapters | Process starts, lints, type-checks, tests green |

---

## 5. Scope

### In scope (v0.1 — the minimal viable multi-platform bot)

- `prismal_chatbot` package: platform-agnostic `ConversationBridge` that owns
  the entire `prismal-sdk` turn lifecycle (thread resolution, streaming
  consumption, error mapping) independent of which platform triggered it.
- Slack adapter (`slack_bolt`, **Socket Mode** — no public inbound HTTP
  endpoint required): responds to `app_mention` and direct messages, posts a
  placeholder reply and edits it as the bridge streams, replies in-thread.
- Discord adapter (`discord.py`, Gateway): responds to bot mentions and DMs,
  posts a placeholder reply and edits it as the bridge streams.
- `ThreadStore`: local SQLite-backed persistence mapping
  `(platform, external_conversation_key) -> thread_id`, so restarts resume
  the same engine conversation.
- Streaming throttle: token deltas are coalesced and flushed on a bounded
  cadence (time- and/or char-based) per platform's edit-rate budget; the
  final `DoneEvent` always flushes the exact final text.
- Tool-activity indicator: rendered as a transient "🔧 using web_search…"
  edit, replaced by the next content flush (best-effort — chat platforms
  have no dedicated activity affordance).
- Error UX: typed SDK exceptions mapped to a friendly platform message,
  posted as a reply (no retry button — platform UX is "just ask again").
- Cancellation: a bounded per-turn timeout guard (`PRISMAL_CHATBOT_TURN_TIMEOUT_S`)
  breaks the stream and posts a friendly timeout message if the engine never
  reaches `DoneEvent` — chat platforms have no "stop" affordance equivalent
  to the webchat's stop button.
- Configuration via `PRISMAL_CHATBOT_` environment variables (server
  connection) plus per-platform variables (`PRISMAL_CHATBOT_SLACK_*`,
  `PRISMAL_CHATBOT_DISCORD_*`); each adapter is independently enabled/disabled.
- Single-workspace / single-guild credentials per deployment in v0.1 (one
  Slack app install, one Discord bot token) — see Out of scope for
  multi-tenant install flows.
- Unit test suite offline (fake SDK client + fake platform SDK clients);
  opt-in `contract` suite against a running `prismal-server`.
- `uv`-based dev foundation: `pyproject.toml`, `ruff`, `mypy`, `pytest` +
  `pytest-asyncio`, CI workflow; Dockerfile for one-container deploy running
  both adapters as asyncio tasks.

### Out of scope (deferred)

- **Multi-workspace / multi-guild OAuth install flow** ("Add to Slack" /
  Discord OAuth invite with per-installation token storage) — v0.1 targets
  one workspace and one guild per deployment via static bot tokens; a
  self-serve install flow is CHB-FUTURE-01.
- **Telegram, WhatsApp, MS Teams adapters** — the presentation and initial
  scope name Slack/Discord only; the `ConversationBridge` is designed
  platform-agnostic precisely so additional adapters are additive
  (CHB-FUTURE-02).
- **Slash commands / interactive components** (buttons, modals, Discord
  application commands) — v0.1 responds to mentions and DMs only;
  interactive UX is CHB-FUTURE-03.
- **File/media upload or output** — the engine's multimodal layer is opt-in
  and the chat contract carries text `content` only in v0.1, matching
  `prismal-webchat`'s same deferral; revisit when the host specs a media
  route (CHB-FUTURE-04).
- **Per-user identity / DID mapping** (mapping a Slack/Discord user to an
  engine-side identity or per-user budget) — v0.1 authenticates the bot
  process to `prismal-server` with one service credential per platform
  deployment; per-user tenancy is CHB-FUTURE-05.
- **A2A surface** — like the webchat, this repo is a chat consumer; it does
  not expose or consume A2A. Generic A2A peers already talk to
  `prismal-server` directly.
- **Horizontal scale-out** (multiple bot replicas sharing one `ThreadStore`,
  distributed locking) — single-process SQLite deploy in v0.1; a shared
  store (e.g. Postgres) is CHB-FUTURE-06 if real load demands it.

---

## 6. Functional Requirements (summary; refined in `SPEC.md`)

| ID | Requirement | Priority |
|---|---|---|
| RF-CHB-001 | Platform-agnostic `ConversationBridge` owning the `prismal-sdk` turn lifecycle (thread resolution, streaming consumption, error mapping), reusable by any adapter | `MUST` |
| RF-CHB-002 | Slack adapter (Socket Mode): mention/DM triggers, threaded placeholder reply, throttled edits | `MUST` |
| RF-CHB-003 | Discord adapter (Gateway): mention/DM triggers, placeholder reply, throttled edits | `MUST` |
| RF-CHB-004 | `ThreadStore`: durable `(platform, external_conversation_key) -> thread_id` mapping surviving process restarts | `MUST` |
| RF-CHB-005 | Streaming throttle bounding edit frequency per platform, with a guaranteed final flush on `DoneEvent` | `MUST` |
| RF-CHB-006 | All `prismal-server` communication goes through `prismal_sdk`; zero wire-contract code and zero `prismal`/`prismal_server` imports here | `MUST` |
| RF-CHB-007 | Per-platform configuration (`PRISMAL_CHATBOT_SLACK_*` / `PRISMAL_CHATBOT_DISCORD_*`) with independent enable/disable; all secrets as `SecretStr` | `MUST` |
| RF-CHB-008 | Error UX: typed SDK exceptions mapped to a friendly platform message; per-turn timeout guard when the engine never completes | `MUST` |
| RF-CHB-009 | Offline unit suite (fake SDK client + fake platform clients) + opt-in `contract` suite against a live host | `MUST` |

---

## 7. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Scope creep — the bot grows agent/business logic (prompting, retries, RAG) | `CLAUDE.md` hard rule + review checklist: platform adapters and session mapping only; the engine owns intelligence, the SDK owns transport |
| Wire-contract drift reaches the bot | The SDK absorbs contract changes; this repo pins `prismal-sdk>=0.1,<1` and upgrades deliberately; `contract` suite catches breakage |
| Platform edit-rate limits (429s) from naive token-by-token streaming | Coalescing throttle in `ConversationBridge` (§4, `ARCHITECTURE.md`); unit-tested edit-count bound |
| Lost conversation continuity on restart | `ThreadStore` (SQLite) persists the mapping before the first `create_thread()` call returns to the adapter |
| Bot token leaks (Slack/Discord tokens, `prismal-server` token) | All platform + server credentials as `SecretStr`; a dedicated test asserts no secret appears in logs |
| One adapter's outage takes down the other | Adapters run as independent asyncio tasks with isolated exception handling; one adapter crashing logs and exits its task without killing the process (documented restart policy in `docs/deploy.md`) |
| Single shared service credential per platform = one shared budget/tenant | Acceptable for v0.1 (documented); per-user/per-workspace tenancy is CHB-FUTURE-05, and the engine's budget/tenancy layers already exist server-side |
| SQLite `ThreadStore` under concurrent adapters in one process | Single-writer access pattern (asyncio-serialized), WAL mode; multi-replica sharing is explicitly out of scope (CHB-FUTURE-06) |

---

## 8. Dependencies

- `prismal-sdk` — the **only** path to `prismal-server`. Typed models,
  streaming events, exception hierarchy. No other prismal dependency.
- `slack-bolt` (Socket Mode) — Slack platform adapter.
- `discord.py` — Discord platform adapter.
- Dev-only: `pytest`, `pytest-asyncio`, `ruff`, `mypy`.
- A running `prismal-server` for `contract` tests and manual dev (local
  `uvicorn` or Docker; see that repo's docs); a Slack app / Discord
  application for manual platform testing.
- **Not** dependencies: `prismal` (engine), `prismal_server` (host), `httpx`
  as a direct dep against `prismal-server` (it arrives transitively via the
  SDK and must not be used directly here — enforced by the boundary test).

---

## 9. Milestones

| Milestone | Content | Exit criterion |
|---|---|---|
| M0 — Seed | This SDD set (`PLAN`/`ARCHITECTURE`/`SPEC`/`TASKS`) + repo `CLAUDE.md` + `README.md` | Reviewed & merged |
| M1 — App skeleton | `uv` scaffold, settings per platform, boundary guard test, `ThreadStore` schema, CI | Process boots with both adapters disabled by default; guards green |
| M2 — Bridge core | `ConversationBridge` turn lifecycle against a fake SDK client + fake platform sink; thread resolution/persistence | Unit-tested turn: thread reuse across a simulated restart, throttled flush, done/error terminal states |
| M3 — Slack adapter | Socket Mode connection, mention/DM handling, threaded placeholder + edits | Manual verification against a real Slack workspace; unit tests offline via fake Bolt client |
| M4 — Discord adapter | Gateway connection, mention/DM handling, placeholder + edits | Manual verification against a real Discord guild; unit tests offline via fake discord.py client |
| M5 — Hardening & release | `contract` suite vs a real host, secret-leak test, Dockerfile (both adapters as tasks), docs, `v0.1.0` tag | CI green; one-container deploy answers in a live Slack workspace and Discord guild against a live `prismal-server` |

---

## Change History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-07-19 | Ernesto Crespo | Initial PLAN for the external bots (Slack/Discord) front-end, derived from the ecosystem presentation (v3.10.1), the `prismal-server` wire contract, the `prismal-sdk` client surface, and the pattern established in `prismal-webchat` |
