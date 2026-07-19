# prismal-chatbot — Technical Specification (Chatbot Bootstrap)

| Field | Value |
|---|---|
| **Status** | `DRAFT` |
| **Version** | 1.0 |
| **Date** | 2026-07-19 |
| **Companion** | [`PLAN.md`](./PLAN.md) · [`ARCHITECTURE.md`](./ARCHITECTURE.md) · [`TASKS.md`](./TASKS.md) |

Requirement IDs are `SPEC-CHB-<AREA>-NNN`. Priority follows RFC 2119
(`MUST`/`SHOULD`/`MAY`). Each maps back to an `RF-CHB-*` in `PLAN.md`. Where a
requirement leans on a sibling contract, the sibling's ID is cited
(`SPEC-CSB-*` from `prismal-sdk`, `SPEC-RHB-*` from `prismal-server`).

---

## 1. Application & construction

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-APP-001 | The process MUST expose one entrypoint (`python -m prismal_chatbot`) that constructs a single `ConversationBridge`, a single `AsyncPrismalClient`, a single `ThreadStore`, and starts one `asyncio` task per **enabled** platform adapter (`asyncio.gather`, tasks isolated so one adapter's crash does not kill the process, `ADR-004`). | `MUST` |
| SPEC-CHB-APP-002 | The repo MUST NOT import `prismal` or `prismal_server`, and MUST NOT use `httpx` (or any HTTP client) directly against `prismal-server` — all upstream traffic goes through `prismal_sdk`. Enforced by an AST boundary-guard test. | `MUST` |
| SPEC-CHB-APP-003 | The SDK client MUST be a process-wide `AsyncPrismalClient` built lazily from `ChatbotSettings` (`client.py::get_client()`), closed on process shutdown. | `MUST` |
| SPEC-CHB-APP-004 | At least one platform adapter MUST be enabled at startup; if none are enabled the process MUST fail fast with a clear error rather than start idle. | `MUST` |
| SPEC-CHB-APP-005 | Platform SDK imports (`slack_bolt`, `discord.py`) MUST stay isolated inside `platforms/*_adapter.py`; `bridge/*` and `store/*` MUST NOT import a platform SDK (keeps the bridge unit-testable without either platform installed at test time — mirrors the engine's provider-isolation rule). | `MUST` |

## 2. Conversation bridge & threading

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-BRG-001 | `ConversationBridge.handle_turn(key, text, responder)` MUST resolve `thread_id` via `ThreadStore.get(key)`; on miss it MUST call `create_thread()` exactly once and persist the mapping via `ThreadStore.put(key, thread_id)` **before** issuing `send_message()`. | `MUST` |
| SPEC-CHB-BRG-002 | The bridge MUST consume `send_message()`'s typed events and drive a `StreamThrottle`: `TokenEvent.delta` is fed to the throttle (not rendered directly); `ToolCallEvent.name` MAY render a transient activity update; `DoneEvent` MUST force a final throttle flush followed by `responder.finalize(final_text)`. `StateEvent` MAY be ignored (logged at debug). | `MUST` |
| SPEC-CHB-BRG-003 | At most one in-flight turn per `ConversationKey` MUST be processed at a time; a new message for a key with an in-flight turn MUST be queued or rejected with a friendly "still working on your last message" reply (implementation choice recorded in `TASKS.md`; either satisfies this requirement). | `MUST` |
| SPEC-CHB-BRG-004 | Every turn MUST be bounded by `PRISMAL_CHATBOT_TURN_TIMEOUT_S`; on expiry the bridge MUST cancel stream consumption promptly (SDK closes the stream → host cancels the run, `SPEC-CSB-CHT-006` → `SPEC-RHB-CHT-004`) and call `responder.error(...)` with a timeout-specific message. | `MUST` |
| SPEC-CHB-BRG-005 | User-provided text MUST be passed to `send_message(content=…)` unmodified — no templating, prefixing, or prompt construction anywhere in this repo (`SPEC-CSB-CHT-007`; the engine's security layers apply server-side). | `MUST` |
| SPEC-CHB-BRG-006 | `ConversationKey` construction MUST be deterministic and platform-documented (§4.4 `ARCHITECTURE.md`): Slack keys include the thread timestamp when replying in-thread; Discord keys distinguish guild channels from DMs. | `MUST` |

## 3. Streaming throttle

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-THR-001 | `StreamThrottle` MUST flush an accumulated-text update to the platform no more often than once per configured `edit_interval_s`, except the mandatory final flush on `DoneEvent`/error/timeout. | `MUST` |
| SPEC-CHB-THR-002 | `StreamThrottle` MAY also flush early once accumulated new content exceeds a configured `edit_min_chars`, to keep visible latency reasonable on slow-interval configs. | `SHOULD` |
| SPEC-CHB-THR-003 | A scripted long token stream (unit test) MUST produce a bounded number of platform update calls proportional to `turn_duration / edit_interval_s`, not to token count. | `MUST` |
| SPEC-CHB-THR-004 | The final rendered text after `DoneEvent` MUST equal the full concatenation of all `TokenEvent.delta`s, regardless of throttling (no content loss). | `MUST` |

## 4. Persistence (`ThreadStore`)

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-STO-001 | `ThreadStore` MUST persist `(platform, external_conversation_key) -> thread_id` in SQLite with WAL mode enabled, at a path from `PRISMAL_CHATBOT_STORE_PATH`. | `MUST` |
| SPEC-CHB-STO-002 | `ThreadStore.get`/`put` MUST be safe for concurrent use by multiple adapter tasks in the same process (single-writer serialization internally). | `MUST` |
| SPEC-CHB-STO-003 | A process restart MUST resolve a previously-seen `ConversationKey` to the same `thread_id` (round-trip unit test using a real temp SQLite file, not a fake). | `MUST` |
| SPEC-CHB-STO-004 | The store schema MUST be created automatically on first run if the file/table does not exist (no manual migration step for v0.1). | `MUST` |

## 5. Platform adapters

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-SLK-001 | `SlackAdapter` MUST connect via Socket Mode (no public inbound HTTP endpoint required) and handle `app_mention` events and direct messages. | `MUST` |
| SPEC-CHB-SLK-002 | On receiving a triggering event, the adapter MUST post a placeholder reply in-thread (creating a thread on the triggering message if none exists) before invoking the bridge, and implement `Responder` by editing that placeholder via `chat.update`. | `MUST` |
| SPEC-CHB-SLK-003 | The adapter MUST ignore messages from bots (including itself) to avoid response loops. | `MUST` |
| SPEC-CHB-DSC-001 | `DiscordAdapter` MUST connect via the Gateway (`discord.py`) with the `message_content` intent, and handle bot mentions in guild channels and DMs. | `MUST` |
| SPEC-CHB-DSC-002 | On receiving a triggering event, the adapter MUST post a placeholder reply before invoking the bridge, and implement `Responder` by editing that message. | `MUST` |
| SPEC-CHB-DSC-003 | The adapter MUST ignore messages from bots (including itself) to avoid response loops. | `MUST` |
| SPEC-CHB-PLT-001 | Both adapters MUST implement the same `PlatformAdapter`/`Responder` Protocols (`platforms/base.py`); the bridge MUST NOT branch on platform identity. | `MUST` |

## 6. Error UX

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-ERR-001 | Every `PrismalSDKError` raised during a turn MUST be caught in `ConversationBridge.handle_turn()` and mapped to platform-agnostic friendly text (`errors.py`), then posted via `responder.error(...)`. | `MUST` |
| SPEC-CHB-ERR-002 | Error codes, server messages, and stack traces MUST NOT be posted to the platform; they go to bot logs only (mirrors `SPEC-RHB-ERR-002` / `SPEC-WCB-ERR-002`). | `MUST` |
| SPEC-CHB-ERR-003 | A mid-stream error MUST preserve whatever partial text was already flushed to the platform message, appending a short error note rather than replacing it. | `MUST` |
| SPEC-CHB-ERR-004 | There is no automatic retry of a failed turn; a subsequent user message is treated as a new turn on the same (already-resolved) `thread_id`. | `MUST` |

## 7. Auth, tenancy & secret hygiene

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-AUT-001 | The bot authenticates to `prismal-server` with a single service credential from `ChatbotSettings.token` (`SecretStr`), passed only to the SDK client constructor. | `MUST` |
| SPEC-CHB-AUT-002 | Slack/Discord bot tokens MUST be `SecretStr` fields, read once at adapter startup, and MUST NOT appear in logs, error text, or any message posted to a platform. A dedicated test MUST assert their absence from log output. | `MUST` |
| SPEC-CHB-AUT-003 | When `org_id` is configured it MUST be applied client-wide via the SDK (`X-Org-Id`, `SPEC-CSB-AUT-002`); the bot MUST NOT invent any other header or tenancy mechanism. | `MUST` |

## 8. Configuration

`ChatbotSettings` (pydantic-settings, prefix `PRISMAL_CHATBOT_`) with nested
`slack` / `discord` sub-settings; all tokens are backend-only `SecretStr`.

| Key | Env var | Default | Meaning |
|---|---|---|---|
| `server_url` | `PRISMAL_CHATBOT_SERVER_URL` | `http://localhost:8000` | Target `prismal-server` |
| `token` | `PRISMAL_CHATBOT_TOKEN` | `None` | Service bearer credential (`SecretStr`) |
| `org_id` | `PRISMAL_CHATBOT_ORG_ID` | `None` | Tenant for all bot traffic |
| `turn_timeout_s` | `PRISMAL_CHATBOT_TURN_TIMEOUT_S` | `120` | Per-turn bound (§2, `BRG-004`) |
| `store_path` | `PRISMAL_CHATBOT_STORE_PATH` | `./data/thread_store.db` | `ThreadStore` SQLite file |
| `slack.enabled` | `PRISMAL_CHATBOT_SLACK_ENABLED` | `false` | Enable the Slack adapter |
| `slack.bot_token` | `PRISMAL_CHATBOT_SLACK_BOT_TOKEN` | `None` | Slack bot token (`xoxb-…`, `SecretStr`) |
| `slack.app_token` | `PRISMAL_CHATBOT_SLACK_APP_TOKEN` | `None` | Slack app-level token for Socket Mode (`xapp-…`, `SecretStr`) |
| `slack.edit_interval_s` | `PRISMAL_CHATBOT_SLACK_EDIT_INTERVAL_S` | `1.0` | Throttle interval (§3) |
| `discord.enabled` | `PRISMAL_CHATBOT_DISCORD_ENABLED` | `false` | Enable the Discord adapter |
| `discord.bot_token` | `PRISMAL_CHATBOT_DISCORD_BOT_TOKEN` | `None` | Discord bot token (`SecretStr`) |
| `discord.edit_interval_s` | `PRISMAL_CHATBOT_DISCORD_EDIT_INTERVAL_S` | `1.0` | Throttle interval (§3) |

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-CFG-001 | `server_url` MUST be validated as an absolute `http(s)://` URL at startup, failing fast. | `MUST` |
| SPEC-CHB-CFG-002 | If `slack.enabled=true`, `slack.bot_token` and `slack.app_token` MUST both be present or startup MUST fail fast with a clear error naming the missing variable(s). Same rule for `discord.enabled=true` and `discord.bot_token`. | `MUST` |
| SPEC-CHB-CFG-003 | `edit_interval_s` values MUST be validated as positive floats at startup. | `MUST` |

## 9. Non-functional

| ID | Requirement | Prio |
|---|---|---|
| SPEC-CHB-NFR-001 | The unit suite MUST run with no network and no live platform connection, using `FakeAsyncPrismalClient` plus fake Slack/Discord clients (`ADR-006`); a separate opt-in `contract` suite exercises a live `prismal-server` through the bridge. | `MUST` |
| SPEC-CHB-NFR-002 | `ruff check .` and `mypy` on `prismal_chatbot/` MUST be clean in CI. | `MUST` |
| SPEC-CHB-NFR-003 | Logs MUST NOT contain message content or any secret; they MAY contain platform, a hashed/truncated `external_conversation_key`, `thread_id`, event counts, edit counts, and mapped error codes. | `MUST` |
| SPEC-CHB-NFR-004 | One adapter task raising an unhandled exception MUST be logged and MUST NOT terminate the other adapter task or the process (isolated task supervision, `ADR-004`). | `MUST` |
| SPEC-CHB-NFR-005 | Version pins: `prismal-sdk>=0.1,<1`; `slack-bolt` and `discord.py` pinned to a tested minor; bumps are deliberate PRs, not drive-by. | `MUST` |

---

## 10. Traceability (RF → SPEC)

| RF | Covered by |
|---|---|
| RF-CHB-001 | SPEC-CHB-BRG-001..006 |
| RF-CHB-002 | SPEC-CHB-SLK-001..003 |
| RF-CHB-003 | SPEC-CHB-DSC-001..003 |
| RF-CHB-004 | SPEC-CHB-STO-001..004 |
| RF-CHB-005 | SPEC-CHB-THR-001..004 |
| RF-CHB-006 | SPEC-CHB-APP-002/003/005 |
| RF-CHB-007 | SPEC-CHB-CFG-001..003, SPEC-CHB-AUT-001..003 |
| RF-CHB-008 | SPEC-CHB-ERR-001..004, SPEC-CHB-BRG-004 |
| RF-CHB-009 | SPEC-CHB-NFR-001 |

---

## Change History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-07-19 | Ernesto Crespo | Initial technical spec for the external bots (Slack/Discord) front-end |
