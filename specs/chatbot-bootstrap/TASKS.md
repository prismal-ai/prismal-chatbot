# prismal-chatbot — Task Breakdown (Chatbot Bootstrap)

| Field | Value |
|---|---|
| **Status** | `DRAFT` |
| **Version** | 1.0 |
| **Date** | 2026-07-19 |
| **Companion** | [`PLAN.md`](./PLAN.md) · [`ARCHITECTURE.md`](./ARCHITECTURE.md) · [`SPEC.md`](./SPEC.md) |

Status legend: `TODO` · `WIP` · `DONE` · `BLOCKED`. Every task is **test-first
(TDD)** — write the failing test, watch it fail, minimal code to green. Tasks
map to `SPEC-CHB-*` IDs and the `M0…M5` milestones in `PLAN.md`.

---

## Phase 0 — Seed & scaffold (M0/M1)

| ID | Task | Est. | Dep | SPEC | Status |
|---|---|---|---|---|---|
| CHB-00-01 | SDD set (`PLAN`/`ARCHITECTURE`/`SPEC`/`TASKS`) + `CLAUDE.md` + `README.md` | 0.5 d | — | — | `DONE` |
| CHB-00-02 | `pyproject.toml` (`prismal-sdk>=0.1,<1`, `slack-bolt`, `discord.py`; dev extras `pytest`, `pytest-asyncio`, `ruff`, `mypy`) | 0.4 d | 00-01 | NFR-005 | `TODO` |
| CHB-00-03 | `settings.py`: `ChatbotSettings` + nested `SlackSettings`/`DiscordSettings` (prefix, `SecretStr` tokens, URL/interval validation, conditional-required fail-fast) | 0.5 d | 00-02 | CFG-001..003, AUT-001/002 | `TODO` |
| CHB-00-04 | Boundary-guard test: AST scan — no `prismal`/`prismal_server` import, no direct `httpx` use, no platform SDK import outside `platforms/` | 0.3 d | 00-02 | APP-002/005 | `TODO` |
| CHB-00-05 | `tests/conftest.py`: `FakeAsyncPrismalClient` (scripted event streams), `FakeSlackClient`, `FakeDiscordClient`, tmp `ThreadStore` fixture | 0.5 d | 00-02 | NFR-001 | `TODO` |
| CHB-00-06 | CI (`.github/workflows/ci.yml`): ruff + mypy + pytest (unit); opt-in `contract` job | 0.3 d | 00-02 | NFR-002 | `TODO` |

## Phase 1 — Bridge core (M2)

| ID | Task | Est. | Dep | SPEC | Status |
|---|---|---|---|---|---|
| CHB-01-01 | `client.py`: lazy process-wide `AsyncPrismalClient` from settings; closed on shutdown | 0.3 d | 00-03 | APP-003 | `TODO` |
| CHB-01-02 | `store/thread_store.py`: SQLite `ThreadStore` (WAL, auto-create schema), `get`/`put`, concurrency-safe | 0.6 d | 00-02 | STO-001/002/004 | `TODO` |
| CHB-01-03 | Restart-continuity test: real temp SQLite file, same key resolves to same `thread_id` across two `ThreadStore` instances | 0.3 d | 01-02 | STO-003 | `TODO` |
| CHB-01-04 | `bridge/throttle.py`: `StreamThrottle` (interval + min-chars flush, mandatory final flush) | 0.5 d | 00-02 | THR-001..004 | `TODO` |
| CHB-01-05 | `bridge/conversation_bridge.py`: `handle_turn()` — thread resolution before first send, streaming loop driving the throttle, `DoneEvent` → `finalize()` | 0.8 d | 01-01, 01-02, 01-04, 00-05 | BRG-001/002/005/006 | `TODO` |
| CHB-01-06 | Per-key in-flight guard (queue or reject on concurrent turn for the same `ConversationKey`) | 0.4 d | 01-05 | BRG-003 | `TODO` |
| CHB-01-07 | Turn timeout: cancel stream consumption on `PRISMAL_CHATBOT_TURN_TIMEOUT_S` expiry, friendly timeout error | 0.4 d | 01-05 | BRG-004 | `TODO` |
| CHB-01-08 | `errors.py`: SDK exception → friendly-text mapping table + unit tests for each mapped type | 0.4 d | 01-05 | ERR-001..004 | `TODO` |

## Phase 2 — Slack adapter (M3)

| ID | Task | Est. | Dep | SPEC | Status |
|---|---|---|---|---|---|
| CHB-02-01 | `platforms/base.py`: `PlatformAdapter` / `Responder` Protocols | 0.3 d | 01-05 | PLT-001 | `TODO` |
| CHB-02-02 | `platforms/slack_adapter.py`: Socket Mode connection, `app_mention` + DM handlers, bot-message filtering | 0.7 d | 02-01, 00-03 | SLK-001/003 | `TODO` |
| CHB-02-03 | Slack `Responder`: placeholder reply in-thread + `chat.update` edits wired to `StreamThrottle` | 0.5 d | 02-02, 01-04 | SLK-002 | `TODO` |
| CHB-02-04 | Slack adapter unit tests against `FakeSlackClient` (no live socket) | 0.5 d | 02-03, 00-05 | NFR-001 | `TODO` |
| CHB-02-05 | Manual verification checklist against a real Slack workspace (`docs/slack-setup.md`) | 0.3 d | 02-04 | — | `TODO` |

## Phase 3 — Discord adapter (M4)

| ID | Task | Est. | Dep | SPEC | Status |
|---|---|---|---|---|---|
| CHB-03-01 | `platforms/discord_adapter.py`: Gateway connection (`message_content` intent), mention + DM handlers, bot-message filtering | 0.7 d | 02-01, 00-03 | DSC-001/003 | `TODO` |
| CHB-03-02 | Discord `Responder`: placeholder reply + message edits wired to `StreamThrottle` | 0.5 d | 03-01, 01-04 | DSC-002 | `TODO` |
| CHB-03-03 | Discord adapter unit tests against `FakeDiscordClient` (no live gateway) | 0.5 d | 03-02, 00-05 | NFR-001 | `TODO` |
| CHB-03-04 | Manual verification checklist against a real Discord guild (`docs/discord-setup.md`) | 0.3 d | 03-03 | — | `TODO` |
| CHB-03-05 | `__main__.py`: wire both adapters behind their `enabled` flags, `asyncio.gather` with isolated task supervision, fail-fast if none enabled | 0.5 d | 02-02, 03-01 | APP-001/004, NFR-004 | `TODO` |

## Phase 4 — Hardening & release (M5)

| ID | Task | Est. | Dep | SPEC | Status |
|---|---|---|---|---|---|
| CHB-04-01 | Secret-leak test: platform tokens + server token absent from logs, error text, and any posted message | 0.4 d | 02-02, 03-01, 01-08 | AUT-002, NFR-003 | `TODO` |
| CHB-04-02 | `tests/contract/`: opt-in suite — bridge vs a running `prismal-server` (thread reuse, streamed turn, error path), platform layer still faked | 0.5 d | all Phase 1 | NFR-001 | `TODO` |
| CHB-04-03 | Dockerfile (both adapter tasks, `ThreadStore` on a mounted volume) + `docs/deploy.md` (restart policy, volume note) | 0.5 d | 03-05 | — | `TODO` |
| CHB-04-04 | `docs/configuration.md`, `docs/slack-setup.md`, `docs/discord-setup.md` | 0.4 d | 02-05, 03-04 | — | `TODO` |
| CHB-04-05 | `ruff` + `mypy` clean; coverage baseline recorded | 0.3 d | all | NFR-002 | `TODO` |
| CHB-04-06 | Tag `v0.1.0`; release workflow (GHCR image optional) | 0.2 d | 04-01..05 | — | `TODO` |

## Future (out of scope for v0.1, tracked only)

| ID | Task | Trigger |
|---|---|---|
| CHB-FUTURE-01 | Multi-workspace/multi-guild OAuth install flow, per-installation token storage | A deployment needs to serve more than one workspace/guild |
| CHB-FUTURE-02 | Telegram / WhatsApp / MS Teams adapters | Real demand; `PlatformAdapter` seam already supports it additively |
| CHB-FUTURE-03 | Slash commands / interactive components (buttons, modals, Discord application commands), incl. an explicit `/stop` | Real demand for interactive UX |
| CHB-FUTURE-04 | File/media upload to the multimodal layer | Host specs a media upload route (same trigger as `prismal-webchat` WCB-FUTURE-04) |
| CHB-FUTURE-05 | Per-user identity / DID mapping and per-user budget | A deployment needs per-user tenancy/limits |
| CHB-FUTURE-06 | Shared/external `ThreadStore` (e.g. Postgres) for multi-replica deploys | Real load or HA requirement on a production deployment |

---

## Definition of Done (v0.1)

- `uv pip install -e ".[dev]"` + `python -m prismal_chatbot` boots with at
  least one platform enabled; `ruff check .` and `mypy prismal_chatbot` are
  clean.
- Mentioning the bot (or DMing it) in a connected Slack workspace and in a
  connected Discord guild produces a streamed, throttled, edited reply —
  verified offline against fakes and manually against real workspaces/guilds.
- A conversation's `thread_id` survives a process restart (`ThreadStore`
  round-trip test with a real SQLite file).
- A scripted long token stream produces a bounded number of platform edit
  calls, and the final text is never truncated relative to the full
  concatenation of deltas.
- The boundary-guard test proves no `prismal`/`prismal_server` import and no
  direct HTTP use against the host; the secret-leak test proves no token
  (platform or server) ever reaches a log line or a posted message.
- The `contract` suite passes against a locally running `prismal-server`.
- One-container Docker deploy answers in both a live Slack workspace and a
  live Discord guild against a live `prismal-server`, and a container
  restart continues existing conversations.

## Estimate roll-up

~**12 person-days** across M1–M5 (excludes the M0 seed). Critical path is
Phase 0 → 1 → 2 → 3 → 4 (scaffold before bridge core before either adapter
before hardening); Phases 2 and 3 (Slack, Discord) can run in parallel once
Phase 1 is `DONE`, since both depend only on the bridge and
`platforms/base.py`, not on each other. Only CHB-00-01 (this SDD seed) is
`DONE`; all implementation is `TODO` — the repo is currently `PENDIENTE` per
the ecosystem status.

---

## Change History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-07-19 | Ernesto Crespo | Initial task breakdown for the external bots (Slack/Discord) front-end |
