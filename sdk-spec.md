# DMZAgent SDK Specification

**Version:** see [`VERSION`](./VERSION) — currently `0.6.0`
**Status:** pre-1.0 (MINOR bumps may include breaking wire changes)
**Last updated:** 2026-06-13

This document defines the public surface every DMZAgent SDK MUST
implement. Every language binding — Python, TypeScript, C#, Java — is a
translation of this surface. Same constructor shape, same methods (under
language-idiomatic naming), same return types, same error hierarchy,
same wire protocol.

A spec tag (`v0.5.0`) corresponds 1:1 to a release tag in every SDK
repository. No SDK ships a version the spec hasn't blessed.

---

## 1. Wire protocol

### 1.1 Base URL

Default: `https://api.dmzagent.com`. The constructor MUST accept an
override (used for staging, self-hosted, and local development against
`https://staging.api.eastern-shore-solutions.com` or `http://localhost:8080`).

### 1.2 Authentication

Every request MUST carry:

```
Authorization: Bearer ck_<api-key>
```

API keys start with the prefix `ck_`. The SDK MUST reject empty keys or
keys that do not begin with `ck_` at construction time. Treat key
material as a secret in logs (redact or omit).

Do NOT carry the dashboard session cookie. Do NOT set the legacy
`X-DMZAgent-Key` header — deprecated and slated for removal in v1.0.

### 1.3 Content-Type

All request bodies are JSON-encoded. SDKs MUST set
`Content-Type: application/json` on every POST.

### 1.4 User-Agent

SDKs MUST set:

```
User-Agent: dmzagent-<language>/<spec-version>
```

Examples: `dmzagent-python/0.6.0`, `dmzagent-typescript/0.6.0`,
`dmzagent-csharp/0.6.0`, `dmzagent-java/0.6.0`. Optional suffix in the
form `(<runtime-info>)` is permitted.

### 1.5 Timeout

Default: 10000ms (10 seconds). The constructor MUST expose an override
in the language's native duration type (`float` seconds for Python,
`number` ms for TypeScript, `TimeSpan` for C#, `Duration` for Java).

### 1.6 Ingestion contract

Event ingestion is **accepted-only async** by default: the server stores
the frame, enqueues it for background fan-out reasoning across the
division's workspaces, and returns immediately with a lightweight ack.
Per-workspace reasoning outcomes are retrieved out-of-band via
`await_outcome()` (§5.10), webhooks (§9), or the SDK stream.

The server returns a status field `accepted: true` when the frame was
stored and enqueued, or `accepted: false` when validation or server
capacity rejected it. Do NOT treat `accepted: true` as "reasoning
completed" — it means "the system has the frame and will reason on it."

The legacy synchronous mode (server reasons inline, returns the rich
envelope) is deprecated and will be removed in v1.0.

### 1.7 Reasoning mode

DMZAgent uses **trace-level reasoning** by default. Every ingested
frame is grouped into a trace (§C.1) based on the subject's type and
the configured trace pattern. While a trace is open, frames are stored
and acknowledged but not reasoned immediately. When the trace closes,
reasoning runs once over every frame in the trace as a batch.
`await_outcome()` blocks until the trace closes and batch reasoning
completes.

A **per-frame** mode is available as a premium meter: every frame
triggers immediate background reasoning independently. This is
configured via the division config endpoint (§2.5, §2.6).

The reasoning mode is read from the division config at ingestion time.
SDKs do not set the mode directly; operators configure it through the
division config endpoint.

The `capture()` response is identical in both modes — `accepted: true`
means the frame was stored, not that reasoning finished.

---

## 2. Endpoints

### 2.1 POST /v1/agent-stream/event

Emit one event into the agent stream.

#### Request body

| Field                | Type                | Required | Notes                                                              |
|----------------------|---------------------|----------|--------------------------------------------------------------------|
| `kind`               | enum string         | yes      | `subject_says` \| `tool_call` \| `tool_result` \| `observation`    |
| `agent_subject_id`   | string              | yes      | The conversation anchor — every event grounds against an agent     |
| `payload`            | object              | yes      | May be empty `{}`                                                  |
| `interaction_id`     | string              | no       | Server assigns if omitted; SDK should cache and re-send            |
| `interaction_kind`   | string              | no       | Default `chat_session`                                             |
| `subjects`           | array of subject    | no       | `[{subject_id, role, kind, metadata?}]` — conversation roster     |
| `speaker_subject_id` | string              | no       | The participant who spoke / acted                                  |
| `speaker_role`       | string              | no       | Role label for the speaker                                         |
| `occurred_at`        | string (ISO-8601)   | no       | Defaults to server receive-time                                    |
| `metadata`           | object              | no       | Free-form per-event metadata                                       |

#### Response body

```json
{
  "interaction_id": "int_abc",
  "subjects": ["bot", "cust"],
  "frame_id": "frame_xyz",
  "accepted": true,
  "n_workspaces": 2,
  "follow_my_data": "/v1/frames/frame_xyz/story"
}
```

#### Fields that MAY be absent

When the server can't determine a value, it omits the field rather than
returning `null` or an empty array. The SDK MUST treat missing fields as
"unknown", not as an empty result. `EmitResult` and `CaptureResult` fields
backed by optional response keys MUST be exposed as optional / nullable
in the language.

### 2.2 POST /v1/cb/check

Synchronous circuit-breaker check.

#### Request body

| Field       | Type   | Required | Notes                                       |
|-------------|--------|----------|---------------------------------------------|
| `scope`     | enum   | yes      | `subject` \| `interaction`                  |
| `scope_ref` | string | yes      | The subject_id or interaction_id            |

Exactly one of subject_id / interaction_id must be passed to the SDK
method; the SDK derives `scope` and `scope_ref` from which was set.

#### Response body

```json
{
  "state": "closed",
  "allow": true,
  "warning": false,
  "reason": "no policies fired",
  "fired_policies": [],
  "anchor": null,
  "checked_at": "2026-05-30T12:00:00Z",
  "latency_ms": 12.3,
  "route_latency_ms": 18.7
}
```

`state` ∈ {`closed`, `half_open`, `open`}.
`allow` is `false` only when `state == open`.
`warning` is `true` when `state == half_open`.

`fired_policies` is an array of `{cb_policy_id, name, action}` objects.
`anchor` is an object `{ledger_index, hash}` or `null`.

---

### 2.3 GET /v1/settings/notifications

Fetch the current API key's notification preferences.

#### Response body

```json
{
  "email_cadence": "daily",
  "email_paused_until": null,
  "push_enabled": true,
  "phone": "+14155551234",
  "sms_enabled": false,
  "whatsapp_enabled": true,
  "webhook_url": null
}
```

| Field                | Type                | Notes                                       |
|----------------------|---------------------|---------------------------------------------|
| `email_cadence`      | enum string         | `off` \| `daily` \| `weekly`                |
| `email_paused_until` | string (ISO-8601) \| null | Null when not paused                  |
| `push_enabled`       | boolean             | In-app push notifications                   |
| `phone`              | string \| null      | E.164 phone number for SMS/WhatsApp         |
| `sms_enabled`        | boolean             | SMS delivery enabled for this phone         |
| `whatsapp_enabled`   | boolean             | WhatsApp delivery enabled for this phone    |
| `webhook_url`        | string \| null      | Custom webhook URL for notification delivery|

### 2.4 PUT /v1/settings/notifications

Update the current API key's notification preferences. Supplied fields
are updated; omitted fields are left unchanged.

#### Request body

All fields are optional:

| Field                | Type                | Notes                                       |
|----------------------|---------------------|---------------------------------------------|
| `email_cadence`      | enum string         | `off` \| `daily` \| `weekly`                |
| `email_paused_until` | string (ISO-8601) \| null | Set to null to resume                 |
| `push_enabled`       | boolean             |                                             |
| `phone`              | string              | E.164 format, e.g. `+14155551234`           |
| `sms_enabled`        | boolean             |                                             |
| `whatsapp_enabled`   | boolean             |                                             |
| `webhook_url`        | string \| null      | Custom webhook URL; null to clear           |

#### Response body

Same shape as GET (§2.3) — the complete current set of preferences after
the update is applied.

### 2.5 GET /v1/divisions/{id}/config

Read a division's JSON configuration blob. Used for operator-level
settings such as `reasoning_mode`.

#### Path parameters

| Parameter    | Type   | Notes                                       |
|-------------|--------|---------------------------------------------|
| `id`        | string | Division id, e.g. `div:abc`                 |

#### Response body

```json
{
  "config": {
    "reasoning_mode": "per_trace"
  }
}
```

`config` is an arbitrary JSON object. The server does not enforce a
fixed schema beyond being a valid JSON object. The SDK returns the
object as-is (typed as `Record<string, unknown>` / `dict[str, Any]`).

### 2.6 PUT /v1/divisions/{id}/config

Replace a division's full JSON configuration blob. Requires elevated
permissions (tenant_admin+).

#### Path parameters

Same as GET (§2.5).

#### Request body

```json
{
  "config": {
    "reasoning_mode": "per_trace",
    "custom_setting": 42
  }
}
```

#### Response body

Same shape as GET — the config as stored after replacement.

---

## 3. Error handling

The server returns standard HTTP status codes. Every SDK MUST map them
to a typed exception hierarchy:

| Status | Canonical type      | Meaning                              |
|--------|---------------------|--------------------------------------|
| 400    | `ValidationError`   | malformed payload                    |
| 401    | `AuthError`         | API key missing / invalid / revoked  |
| 403    | `PermissionError`   | key valid but lacks scope            |
| 5xx    | `ServerError`       | transient — safe to retry            |
| other  | `DMZAgentError`    | unexpected status                    |

`CBOpenError` (canonical name) is NOT raised from the HTTP layer. It is
raised by the `guard()` context-manager / using-block when
`raiseOnOpen=true` and the check returns `allow=false`.

Every exception MUST expose:

- `message: string`
- `statusCode: number | null`
- `body: object | string | null`

`CBOpenError` additionally exposes:

- `reason: string`
- `firedPolicies: array<{cb_policy_id, name, action}>`
- `anchor: object | null`
- `scopeRef: string`

### 3.1 Network and timeout

Timeouts and underlying network errors MUST be wrapped as `ServerError`
with a descriptive message. The original cause MUST be preserved via the
language's exception-chaining mechanism (`raise … from e` in Python,
`cause` in TypeScript Error, `InnerException` in C#, `initCause()` in
Java).

---

## 4. Client constructor

Each SDK exposes a primary client class. Canonical name in each
language:

| Language    | Class name          |
|-------------|---------------------|
| Python      | `DMZAgent`         |
| TypeScript  | `DMZAgent`         |
| C#          | `DMZAgentClient`   |
| Java        | `DMZAgentClient`   |

C# and Java diverge from the bare `DMZAgent` name because both
languages reserve unqualified type names for value-bearing entities and
both expect a `Client` / service suffix for HTTP service classes.

### 4.1 Constructor parameters

| Spec name      | Type                 | Required | Default                          |
|----------------|----------------------|----------|----------------------------------|
| `api_key`      | string               | yes      | (none)                           |
| `base_url`     | string               | no       | `https://api.dmzagent.com`      |
| `timeout`      | duration             | no       | 10000ms / 10s                    |
| `user_agent`   | string               | no       | `dmzagent-<lang>/<spec-version>`|

### 4.2 Thread safety

The client MUST be safe to share across threads / async contexts where
the language permits it:

- Python: thread-safe via `httpx.Client` reuse.
- TypeScript: single-loop-per-context, no cross-thread concern.
- C#: `HttpClient` is thread-safe; the wrapping client must be too.
- Java: pick a thread-safe HTTP client (OkHttp recommended).

### 4.3 Resource lifecycle

The client MUST implement the language's idiomatic resource cleanup:

- Python: `__enter__` / `__exit__` (context manager) + explicit
  `close()`.
- TypeScript: explicit `close()` method.
- C#: `IDisposable.Dispose()` + explicit `Close()`.
- Java: `AutoCloseable.close()` + explicit `close()`.

Closing the client MUST release the underlying HTTP transport. Calling
`close()` more than once MUST be a no-op (not an error).

---

## 5. Methods

This section defines the canonical method surface in language-agnostic
snake_case. See §8 for the per-language idiomatic name map.

### 5.1 `emit_event(kind, ...) → EmitResult`

Low-level event emitter. Every higher-level helper delegates to this.

Parameters (snake_case canonical names; language-idiomatic equivalents
per §8):

- `kind: string` — REQUIRED, must be in `EVENT_KINDS`
- `agent_subject_id: string` — REQUIRED
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`. Determines the trace pattern
  (§C.1) used for grouping and deviation evaluation.
- `payload: object` — defaults to `{}`
- `interaction_id?: string`
- `interaction_kind?: string` — default `"chat_session"`
- `subjects?: array<subject>`
- `speaker_subject_id?: string`
- `speaker_role?: string`
- `occurred_at?: string` — ISO-8601
- `metadata?: object`

Validation: throw `ValueError` (Python) / `RangeError` (TS) /
`ArgumentException` (C#) / `IllegalArgumentException` (Java) when
`kind` is not one of `EVENT_KINDS`.

### 5.2 `subject_says(subject_id, text, agent_subject_id, ...) → EmitResult`

Convenience wrapper for `kind = "subject_says"`.

- `subject_id: string` — the speaker, REQUIRED
- `text: string` — REQUIRED
- `agent_subject_id: string` — REQUIRED (conversation anchor)
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`
- `interaction_id?: string`
- `subjects?: array<subject>`
- `payload_extra?: object` — merged into `payload` alongside `{text}`

The SDK passes `speaker_subject_id = subject_id` on the wire.

### 5.3 `tool_call(subject_id, tool, args, ...) → EmitResult`

Convenience wrapper for `kind = "tool_call"`. Use BEFORE the tool runs —
this emits the intent, not the result.

- `subject_id: string` — the agent invoking the tool, REQUIRED
- `tool: string` — REQUIRED
- `args: object` — defaults to `{}`
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`
- `interaction_id?: string`
- `subjects?: array<subject>`

The SDK passes `speaker_role = "agent"` on the wire.

### 5.4 `tool_result(subject_id, tool, result, ...) → EmitResult`

Convenience wrapper for `kind = "tool_result"`. Pair with the prior
`tool_call`.

- `subject_id: string` — REQUIRED
- `tool: string` — REQUIRED
- `result: any` — REQUIRED (the tool's return value; SDK JSON-encodes)
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`
- `interaction_id?: string`
- `subjects?: array<subject>`

### 5.5 `observation(agent_subject_id, subjects, payload, ...) → EmitResult`

Convenience wrapper for `kind = "observation"`. Use for structured
events that don't fit a speech-bubble shape (video keyframes, IoT,
sensor readings).

- `agent_subject_id: string` — REQUIRED
- `subjects: array<subject>` — REQUIRED
- `payload: object` — REQUIRED
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`
- `interaction_id?: string`

### 5.6 `check(subject_id? | interaction_id?) → CheckResult`

Circuit-breaker check. Pass EXACTLY ONE of `subject_id` or
`interaction_id`. Both nil or both set MUST throw a validation error.

### 5.7 `guard(subject_id? | interaction_id?, raise_on_open=false) → context-managed CheckResult`

Resource-scoped wrapper around `check()`. Language idioms:

- Python: `@contextmanager`-style `with cx.guard(...) as g:` yielding
  the `CheckResult`.
- TypeScript: async function returning `{result, [Symbol.dispose]}`
  pair, usable via `using g = cx.guard(...)`. Fallback: callback form
  `cx.guard(opts, async (result) => {...})`.
- C#: `Guard` returns an `IDisposable` wrapper exposing `Result`
  property; usable via `using var g = client.Guard(...);`.
- Java: `Guard` returns an `AutoCloseable` with `getResult()`; usable
  via `try (var g = client.guard(...)) {...}`.

When `raise_on_open = true` and `result.allow == false`, the
language-canonical `CBOpenError` MUST be raised before yielding.

### 5.8 `conversation(participants, agent_subject_id?, kind?, metadata?) → Conversation`

Factory that returns a Conversation handle (see §6).

- `participants: array<participant>` — REQUIRED, non-empty. Each
  participant: `{subject_id, role?, kind?, subject_type?, metadata?}`.
  When `subject_type` is omitted, individual method calls on the
  conversation handle MUST supply it.
- `agent_subject_id?: string` — derived from participants if omitted;
  see §6.1.
- `kind?: string` — interaction_kind, default `"chat_session"`.
- `metadata?: object`.

### 5.9 `capture(subject_id, kind, payload, ...) → CaptureResult`

Ingest a single behavior event and return the accepted-only ack. This is
the primary ingestion method for the accepted-only async contract.

Unlike `emit_event`, `capture` returns a lightweight `CaptureResult`
that does NOT include per-workspace reasoning outcomes — those are
retrieved out-of-band via `await_outcome`, webhooks, or the SDK stream.

Parameters:

- `subject_id: string` — REQUIRED, the subject performing the
  behavior. A free-form slug like `"bot"` or `"cust"`; the server
  resolves it to the API key's division.
- `kind: string` — REQUIRED, must be in `EVENT_KINDS`
- `subject_type: string` — REQUIRED, one of `"chat"`, `"sensor"`,
  `"lead"`, `"ticket"`, `"journey"`. Determines the trace pattern
  (§C.1) used for grouping and deviation evaluation.
- `payload: object` — defaults to `{}`
- `agent_subject_id?: string` — conversation anchor. Required when
  `kind` is `subject_says`, `tool_call`, or `tool_result` and the
  event is part of a tracked interaction.
- `interaction_id?: string`
- `interaction_kind?: string` — default `"chat_session"`
- `subjects?: array<subject>`
- `speaker_subject_id?: string`
- `speaker_role?: string`
- `occurred_at?: string` — ISO-8601
- `metadata?: object`

Validation: throw the language's canonical validation error when `kind`
is not one of `EVENT_KINDS`. Throw a validation error when
`subject_type` is not one of the five allowed values.

### 5.10 `await_outcome(frame_id, timeout?) → OutcomeResult`

Block until reasoning completes for a frame and return the outcome. Polls
the frame story endpoint at a backoff interval (start 100ms, double to
max 2s, cap at `timeout`). Raises `TimeoutError` (language-canonical) if
the timeout is reached before reasoning completes.

This is the **poll variant** of the three retrieval modes (poll, webhook,
stream). Use it when the calling code needs a synchronous-feeling
response and can tolerate up to `timeout` latency.

Parameters:

- `frame_id: string` — returned by `capture().frame_id`
- `timeout?: float` — seconds, default 30.0. MUST cap at 120.0.

Returns `OutcomeResult` (§7.4) with per-workspace reasoning results.

### 5.11 `close() → void`

Release the underlying HTTP client. Idempotent.

### 5.12 `get_notification_prefs() → NotificationPrefs`

Fetch the current API key's notification preferences (§3.1).

Returns `NotificationPrefs` (§7.8).

### 5.13 `update_notification_prefs(...) → NotificationPrefs`

Update notification preferences. Only supplied fields are touched.

Parameters (all optional):

- `email_cadence?: "off" | "daily" | "weekly"`
- `email_paused_until?: string | null` — ISO-8601; null to resume
- `push_enabled?: boolean`
- `phone?: string` — E.164 format
- `sms_enabled?: boolean`
- `whatsapp_enabled?: boolean`
- `webhook_url?: string | null` — null to clear

Returns `NotificationPrefs` (§7.8).

### 5.14 `get_division_config(division_id) → DivisionConfig`

Read a division's configuration blob (§3.3).

- `division_id: string` — REQUIRED

Returns `DivisionConfig` (§7.9).

### 5.15 `update_division_config(division_id, config) → DivisionConfig`

Replace a division's full configuration blob (§3.4). Requires elevated
permissions (tenant_admin+).

- `division_id: string` — REQUIRED
- `config: object` — REQUIRED, arbitrary JSON object

Returns `DivisionConfig` (§7.9).

---

## 6. Conversation handle

A stateful helper for the common single-agent-with-customer case (and
generalizing to multi-agent / multi-subject conversations).

### 6.1 Construction

Conversation is constructed via `client.conversation(...)`. Direct
construction is NOT part of the public API.

`agent_subject_id` is derived from participants when not supplied: first
participant whose `role` (lowercased) is in
`{"agent", "service", "system"}` wins; otherwise the first participant.

`interaction_id` is `null` initially and is captured from the first
`EmitResult.interaction_id` the server returns. Subsequent calls
re-send it so the server stitches events into the same interaction.

### 6.2 Methods on Conversation

| Method                                        | Returns        |
|-----------------------------------------------|----------------|
| `says(subject_id, text, payload_extra?)`      | `EmitResult`   |
| `tool_call(subject_id, tool, args?)`          | `EmitResult`   |
| `tool_result(subject_id, tool, result)`       | `EmitResult`   |
| `observation(payload)`                        | `EmitResult`   |
| `add_subject(subject_id, role?, kind?, metadata?)` | `void`    |
| `check(subject_id)`                           | `CheckResult`  |
| `guard(subject_id, raise_on_open=false)`      | context-managed CheckResult |
| `close()`                                     | `void`         |

Properties:

| Property         | Type             |
|------------------|------------------|
| `interaction_id` | string \| null   |
| `subjects`       | array<subject>   |

`add_subject` is idempotent on `subject_id`: re-adding with the same id
refreshes the role/kind/metadata.

### 6.3 Resource lifecycle

Conversation implements the language's resource idiom (the same one the
client implements). Currently the close path is a no-op; v1.0 will emit
an `end_interaction` event.

---

## 7. Result types

### 7.1 `EmitResult`

Returned by every event-emit method (`emit_event`, `subject_says`,
`tool_call`, `tool_result`, `observation`).

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `interaction_id`  | string                | "" if server omitted                           |
| `subjects`        | array<string>         | subject ids on the resulting frame             |
| `queued`          | boolean               | true (inferred from accepted when absent)      |
| `accepted`        | boolean?              | true when the frame was stored and enqueued    |
| `n_workspaces`    | integer?              | number of workspaces the frame fanned out to   |
| `frame_id`        | string?               | frame id for outcome retrieval                 |
| `follow_my_data`  | string?               | path to the frame story                        |
| `raw`             | object                | the full server JSON response                  |

Legacy fields (`outcome`, `triage_decision`, `tags_fired`,
`soul_version`, `ledger_index`) are no longer populated — those values
are available on the `OutcomeResult` after reasoning completes.

### 7.2 `CaptureResult`

Returned by `capture()`.

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `frame_id`        | string                | frame id for outcome retrieval                 |
| `accepted`        | boolean               | true when the frame was stored and enqueued    |
| `n_workspaces`    | integer               | number of workspaces the frame fanned out to   |
| `interaction_id`  | string                | "" if server omitted                           |
| `subjects`        | array<string>         | subject ids on the resulting frame             |
| `follow_my_data`  | string?               | path to the frame story                        |
| `raw`             | object                | the full server JSON response                  |

### 7.3 `OutcomeResult`

Returned by `await_outcome()`.

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `frame_id`        | string                | the frame that was reasoned                     |
| `outcome`         | string                | `skipped` \| `no_change` \| `applied` \| `failed` |
| `error`           | object?               | `{code, message}` present only when failed     |
| `tags_fired`      | array<tag_fired>      | tags that fired across all workspaces          |
| `reasoning`       | array<trace>          | per-workspace reasoning traces                 |
| `soul_version`    | integer?              | subject's soul version after reasoning         |
| `finished_at`     | string                | ISO-8601 when reasoning completed              |

Where `tag_fired` is:

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `tag_id`          | string                | canonical tag id                               |
| `strength`        | number                | 0.0 – 1.0                                     |
| `family`          | string?               | tag family id                                  |
| `name`            | string?               | human-readable tag name                        |

And `trace` is:

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `workspace_id`    | string                | workspace that produced this trace             |
| `outcome`         | string                | per-workspace outcome                          |
| `tags_proposed`   | array<tag_fired>?     | tags this workspace's reasoning proposed       |
| `error`           | object?               | present only when per-workspace reasoning failed|

### 7.5 `CheckResult`

Returned by `check()`.

| Field              | Type            | Notes                                            |
|--------------------|-----------------|--------------------------------------------------|
| `state`            | enum            | `closed` \| `half_open` \| `open`                |
| `allow`            | boolean         | `false` only when state is `open`                |
| `warning`          | boolean         | `true` when state is `half_open`                 |
| `reason`           | string          |                                                  |
| `fired_policies`   | array<object>   | `[{cb_policy_id, name, action}]`                |
| `anchor`           | object \| null  | `{ledger_index, hash}` or null                  |
| `checked_at`       | string          | ISO-8601, may be ""                              |
| `latency_ms`       | number          | server-side cb.check() latency                   |
| `route_latency_ms` | number          | server-side route handler latency                |
| `raw`              | object          | the full server JSON response                    |

### 7.6 `ReviewEvent`

Webhook payload type for triage/review events delivered via the
coordinate lane (see §9.2).

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `event_id`        | string                | unique event id                                |
| `type`            | string                | `review.opened` \| `review.resolved` \| `review.updated` |
| `review_id`       | string                | the review object id                           |
| `subject_id`      | string                | the subject under review                       |
| `tag_id`          | string                | the tag that fired                             |
| `level`           | string                | `review` \| `escalate`                        |
| `status`          | string                | `open` \| `resolved` \| `dismissed` \| `accepted` |
| `tier`            | string                | `workspace` \| `division` \| `vendor`         |
| `decision`        | string?               | human's decision (present when resolved)       |
| `workspace_id`    | string                | home workspace                                 |
| `division_id`     | string?               | home division                                  |
| `frame_id`        | string?               | triggering frame                               |
| `occurred_at`     | string                | ISO-8601 event timestamp                       |

### 7.8 `NotificationPrefs`

Returned by `get_notification_prefs()` and `update_notification_prefs()`.

| Field                | Type                  | Notes                                     |
|----------------------|-----------------------|-------------------------------------------|
| `email_cadence`      | string                | `off` \| `daily` \| `weekly`             |
| `email_paused_until` | string \| null        | ISO-8601, null when not paused            |
| `push_enabled`       | boolean               | In-app push                               |
| `phone`              | string \| null        | E.164 phone number                        |
| `sms_enabled`        | boolean               | SMS delivery enabled                      |
| `whatsapp_enabled`   | boolean               | WhatsApp delivery enabled                 |
| `webhook_url`        | string \| null        | Custom webhook URL for notification delivery|
| `raw`                | object                | The full server JSON response             |

### 7.9 `DivisionConfig`

Returned by `get_division_config()` and `update_division_config()`.

| Field       | Type                  | Notes                                     |
|-------------|-----------------------|-------------------------------------------|
| `config`    | object                | Arbitrary JSON blob. Empty `{}` when none.|
| `raw`       | object                | The full server JSON response             |

The `config` object is the division's free-form configuration. Canonical
keys recognized by the server include `reasoning_mode` (`"per_frame"` or
`"per_trace"`, see §1.7).

### 7.10 Immutability

Result types MUST be immutable in the language (Python `@dataclass(frozen=True)`,
TypeScript `readonly`, C# `record`, Java `record`).

---

## 8. Naming map

The canonical surface above uses snake_case. Each language's SDK
translates as follows. **No creative liberty** — these are the names
your SDK MUST expose.

### 8.1 Top-level

| Canonical            | Python             | TypeScript         | C#                  | Java               |
|----------------------|--------------------|--------------------|---------------------|--------------------|
| `DMZAgent`          | `DMZAgent`        | `DMZAgent`        | `DMZAgentClient`   | `DMZAgentClient`  |
| `Conversation`       | `Conversation`     | `Conversation`     | `Conversation`      | `Conversation`     |
| `EmitResult`         | `EmitResult`       | `EmitResult`       | `EmitResult` (record) | `EmitResult` (record) |
| `CaptureResult`      | `CaptureResult`    | `CaptureResult`    | `CaptureResult` (record) | `CaptureResult` (record) |
| `OutcomeResult`      | `OutcomeResult`    | `OutcomeResult`    | `OutcomeResult` (record) | `OutcomeResult` (record) |
| `ReviewEvent`        | `ReviewEvent`      | `ReviewEvent`      | `ReviewEvent` (record) | `ReviewEvent` (record) |
| `CheckResult`        | `CheckResult`      | `CheckResult`      | `CheckResult` (record) | `CheckResult` (record) |
| `NotificationPrefs`  | `NotificationPrefs`| `NotificationPrefs`| `NotificationPrefs` (record) | `NotificationPrefs` (record) |
| `DivisionConfig`     | `DivisionConfig`   | `DivisionConfig`   | `DivisionConfig` (record)    | `DivisionConfig` (record)    |
| `EVENT_KINDS`        | `EVENT_KINDS`      | `EVENT_KINDS`      | `EventKinds.All`    | `EventKinds.ALL`   |

### 8.2 Methods

| Canonical          | Python             | TypeScript          | C#                  | Java                |
|--------------------|--------------------|---------------------|---------------------|---------------------|
| `emit_event`       | `emit_event`       | `emitEvent`         | `EmitEvent`         | `emitEvent`         |
| `subject_says`     | `subject_says`     | `subjectSays`       | `SubjectSays`       | `subjectSays`       |
| `tool_call`        | `tool_call`        | `toolCall`          | `ToolCall`          | `toolCall`          |
| `tool_result`      | `tool_result`      | `toolResult`        | `ToolResult`        | `toolResult`        |
| `observation`      | `observation`      | `observation`       | `Observation`       | `observation`       |
| `check`            | `check`            | `check`             | `Check`             | `check`             |
| `guard`            | `guard`            | `guard`             | `Guard`             | `guard`             |
| `capture`          | `capture`          | `capture`           | `Capture`           | `capture`           |
| `await_outcome`    | `await_outcome`    | `awaitOutcome`      | `AwaitOutcome`      | `awaitOutcome`      |
| `conversation`     | `conversation`     | `conversation`      | `Conversation` (method) | `conversation`  |
| `close`            | `close`            | `close`             | `Close` / `Dispose` | `close`             |
| `add_subject`      | `add_subject`      | `addSubject`        | `AddSubject`        | `addSubject`        |
| `get_notification_prefs` | `get_notification_prefs` | `getNotificationPrefs` | `GetNotificationPrefs` | `getNotificationPrefs` |
| `update_notification_prefs` | `update_notification_prefs` | `updateNotificationPrefs` | `UpdateNotificationPrefs` | `updateNotificationPrefs` |
| `get_division_config` | `get_division_config` | `getDivisionConfig` | `GetDivisionConfig` | `getDivisionConfig` |
| `update_division_config` | `update_division_config` | `updateDivisionConfig` | `UpdateDivisionConfig` | `updateDivisionConfig` |

### 8.3 Constructor parameters

| Canonical    | Python       | TypeScript   | C#           | Java         |
|--------------|--------------|--------------|--------------|--------------|
| `api_key`    | `api_key`    | `apiKey`     | `apiKey`     | `apiKey`     |
| `base_url`   | `base_url`   | `baseUrl`    | `baseUrl`    | `baseUrl`    |
| `timeout`    | `timeout`    | `timeout`    | `timeout`    | `timeout`    |
| `user_agent` | `user_agent` | `userAgent`  | `userAgent`  | `userAgent`  |
| `subject_type` | `subject_type` | `subjectType` | `SubjectType` | `subjectType` |

### 8.4 Result fields

| Canonical            | Python              | TypeScript            | C# (record property) | Java (record component) |
|----------------------|---------------------|-----------------------|----------------------|-------------------------|
| `interaction_id`     | `interaction_id`    | `interactionId`        | `InteractionId`      | `interactionId`         |
| `subject_id`         | `subject_id`        | `subjectId`           | `SubjectId`          | `subjectId`             |
| `frame_id`           | `frame_id`          | `frameId`             | `FrameId`            | `frameId`               |
| `accepted`           | `accepted`          | `accepted`            | `Accepted`           | `accepted`              |
| `n_workspaces`       | `n_workspaces`      | `nWorkspaces`         | `NWorkspaces`        | `nWorkspaces`           |
| `outcome`            | `outcome`           | `outcome`             | `Outcome`            | `outcome`               |
| `tags_fired`         | `tags_fired`        | `tagsFired`           | `TagsFired`          | `tagsFired`             |
| `soul_version`       | `soul_version`      | `soulVersion`         | `SoulVersion`        | `soulVersion`           |
| `follow_my_data`     | `follow_my_data`    | `followMyData`        | `FollowMyData`       | `followMyData`          |
| `fired_policies`     | `fired_policies`    | `firedPolicies`       | `FiredPolicies`      | `firedPolicies`         |
| `route_latency_ms`   | `route_latency_ms`  | `routeLatencyMs`      | `RouteLatencyMs`     | `routeLatencyMs`        |
| `error`              | `error`             | `error`               | `Error`              | `error`                 |
| `review_id`          | `review_id`         | `reviewId`            | `ReviewId`           | `reviewId`              |
| `tag_id`             | `tag_id`            | `tagId`               | `TagId`              | `tagId`                 |
| `workspace_id`       | `workspace_id`      | `workspaceId`         | `WorkspaceId`        | `workspaceId`           |
| `division_id`        | `division_id`       | `divisionId`          | `DivisionId`         | `divisionId`            |
| `email_cadence`      | `email_cadence`     | `emailCadence`       | `EmailCadence`       | `emailCadence`          |
| `email_paused_until` | `email_paused_until`| `emailPausedUntil`   | `EmailPausedUntil`   | `emailPausedUntil`      |
| `push_enabled`       | `push_enabled`      | `pushEnabled`        | `PushEnabled`        | `pushEnabled`           |
| `phone`              | `phone`             | `phone`              | `Phone`              | `phone`                 |
| `sms_enabled`        | `sms_enabled`       | `smsEnabled`         | `SmsEnabled`         | `smsEnabled`            |
| `whatsapp_enabled`   | `whatsapp_enabled`  | `whatsappEnabled`    | `WhatsappEnabled`    | `whatsappEnabled`       |
| `webhook_url`        | `webhook_url`       | `webhookUrl`         | `WebhookUrl`         | `webhookUrl`            |
| `config`             | `config`            | `config`             | `Config`             | `config`                |

### 8.5 Exceptions

| Canonical            | Python              | TypeScript            | C#                                       | Java                                    |
|----------------------|---------------------|-----------------------|------------------------------------------|-----------------------------------------|
| `DMZAgentError`     | `DMZAgentError`    | `DMZAgentError`      | `DMZAgentException`                     | `DMZAgentException`                    |
| `AuthError`          | `AuthError`         | `AuthError`           | `DMZAgentAuthException`                 | `DMZAgentAuthException`                |
| `PermissionError`    | `PermissionError`   | `PermissionError`     | `DMZAgentPermissionException`           | `DMZAgentPermissionException`          |
| `ValidationError`    | `ValidationError`   | `ValidationError`     | `DMZAgentValidationException`           | `DMZAgentValidationException`          |
| `ServerError`        | `ServerError`       | `ServerError`         | `DMZAgentServerException`               | `DMZAgentServerException`              |
| `CBOpenError`        | `CBOpenError`       | `CBOpenError`         | `CircuitBreakerOpenException`            | `CircuitBreakerOpenException`           |

C# and Java rename to the `…Exception` convention idiomatic to their
ecosystems. TypeScript follows JS convention with `…Error`. Python keeps
its `…Error` convention.

### 8.6 EVENT_KINDS

The constant array `["subject_says", "tool_call", "tool_result", "observation"]`
MUST be exposed under the language's naming:

- Python: `EVENT_KINDS` (tuple).
- TypeScript: `EVENT_KINDS` (readonly tuple).
- C#: `EventKinds.All` (static IReadOnlyList) plus `EventKinds.SubjectSays`, etc.
- Java: `EventKinds.ALL` (immutable List) plus `EventKinds.SUBJECT_SAYS`, etc.

The on-wire `kind` string is ALWAYS the snake_case form regardless of
how the SDK exposes the enum.

---

## 9. Webhook events

DMZAgent delivers events to registered webhooks for outcomes and
triage/review notifications. Every webhook payload is signed (see §10).

### 9.1 Event envelope

Every webhook POST carries:

```json
{
  "specversion": "1.0",
  "type": "review.opened",
  "source": "/v1/reviews",
  "id": "evt_uuid",
  "time": "2026-06-10T12:00:00Z",
  "datacontenttype": "application/json",
  "data": { ... }
}
```

All standard CloudEvents 1.0 attributes (`specversion`, `type`, `source`,
`id`, `time`, `datacontenttype`) are present. The `data` payload shape
depends on the event type.

### 9.2 Event types

| Type                 | When fired                        | `data` shape            |
|----------------------|-----------------------------------|-------------------------|
| `review.opened`      | A coordinate/review disposition created a new review | `ReviewEvent` (§7.6) |
| `review.resolved`    | A human resolved or dismissed a review | `ReviewEvent` (§7.6) |
| `review.updated`     | A review was claimed, escalated, or reinforced | `ReviewEvent` (§7.6) |
| `outcome.completed`  | Per-workspace reasoning finished for a frame | `OutcomeEvent` |

### 9.3 `OutcomeEvent`

`data` shape for `outcome.completed` events:

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `frame_id`        | string                | the frame that was reasoned                     |
| `workspace_id`    | string                | workspace that produced this outcome            |
| `subject_id`      | string                | subject that was reasoned                       |
| `outcome`         | string                | per-workspace outcome                           |
| `tags_fired`      | array<tag_fired>?     | tags that fired                                 |
| `soul_version`    | integer?              | subject's soul version after reasoning         |
| `ledger_index`    | integer?              | ledger index of the reasoning trace            |

---

## 10. Webhook signature verification

DMZAgent outbound webhooks are signed with HMAC-SHA256 using the
subscription's secret. The signature rides the **`X-DMZAgent-Signature`**
header (value `t=<unix>,v1=<hex>`); the retired **`X-Concordex-Signature`**
header carries the same value for a deprecation window (back-compat, removed
post-launch). Verifiers take the header **value**, so they are unaffected by
the header-name transition. Every SDK MUST expose a helper:

```
verify_webhook_signature(payload, signature_header, secret, tolerance_seconds=300) → bool
```

The verifier MUST:

1. Parse `t=<unix>,v1=<hex>` from the header.
2. Reject (return false / raise) if `t` is missing, non-numeric, or older
   than `tolerance_seconds`.
3. Compute `hmac_sha256(secret, f"{t}.{payload}")` and constant-time
   compare against `v1`.
4. Return `true` only on match.

Reference vectors are in
[`contract-tests/signature-vectors.json`](./contract-tests/signature-vectors.json).
Every SDK's contract test harness MUST run all vectors.

Canonical helper name in each language:

| Language    | Function name                          |
|-------------|----------------------------------------|
| Python      | `verify_webhook_signature` (free fn)   |
| TypeScript  | `verifyWebhookSignature` (free fn)     |
| C#          | `WebhookSignature.Verify` (static)     |
| Java        | `WebhookSignature.verify` (static)     |

---

## 11. Contract tests

Every SDK repository MUST contain a `spec-conformance.yml` GitHub
Actions workflow that:

1. Checks out a pinned tag of this repo.
2. Runs the contract test corpus from `contract-tests/` against the SDK
   build.
3. Reports red/green to the release coordination workflow here (via
   GitHub status check `spec-conformance/<spec-version>`).

The contract test corpus contains:

- [`golden-envelopes.json`](./contract-tests/golden-envelopes.json) —
  fixed inputs → expected wire bodies. Serialization must round-trip
  byte-for-byte (after JSON normalization: sort keys, no trailing
  whitespace).
- [`signature-vectors.json`](./contract-tests/signature-vectors.json) —
  HMAC verification vectors.
- [`error-mapping.json`](./contract-tests/error-mapping.json) — HTTP
  status → exception type the SDK must raise. The harness drives the
  SDK with a fake transport that returns each status and asserts the
  expected exception type.

See [`contract-tests/runner-spec.md`](./contract-tests/runner-spec.md)
for how the harness MUST be wired in each language.

---

## 12. Versioning

The `VERSION` file in this repo is the source of truth. Every SDK
release tag matches this exactly.

Pre-1.0 (current): MINOR bumps MAY include breaking wire changes; PATCH
bumps MUST remain backwards-compatible at both API and wire level.

Post-1.0: MAJOR bumps for breaking changes only, with at least one
MINOR-version deprecation cycle. The deprecation cycle MUST be
documented in `CHANGELOG.md` of the spec repo before a MAJOR is cut.

### 12.1 Spec version pinning

Each SDK pins to a spec version in its language-native manifest:

- Python: `pyproject.toml` → `[tool.dmzagent] spec-version = "0.6.0"`
- TypeScript: `package.json` → `"dmzagent": {"specVersion": "0.6.0"}`
- C#: `Directory.Build.props` → `<DMZAgentSpecVersion>0.6.0</DMZAgentSpecVersion>`
- Java: `pom.xml` → `<dmzagent.spec.version>0.6.0</dmzagent.spec.version>`

The SDK's CI MUST fail-loud if the pinned spec version doesn't match
the version of the spec repo it checks out.

### 12.2 Coordinated release

The `promote.yml` workflow in this repo:

1. Verifies all four SDK repos have a release branch tagged
   `release/v<version>`.
2. Verifies each repo's `spec-conformance` status check is green.
3. Dispatches `publish.yml` to each SDK repo (parallel).
4. Polls each registry (PyPI, npm, NuGet, Maven Central) for the
   artifact.
5. Marks the spec release `promoted` when all four confirm.
6. If any publish fails, runs `yank.yml` to pull artifacts that already
   landed.

---

## 13. Quick-start example (snake_case canonical)

```text
cx = DMZAgent(api_key="ck_…")

# Ingest an event — returns immediately with accepted-only ack.
ack = cx.capture(
    subject_id="bot",
    kind="subject_says",
    subject_type="chat",
    payload={"text": "I want a refund."},
)
print(ack.frame_id, ack.accepted, ack.n_workspaces)

# Retrieve reasoning outcome — blocks until reasoning completes.
outcome = cx.await_outcome(ack.frame_id, timeout=10.0)
print(outcome.outcome, outcome.tags_fired)

# Check circuit breaker before acting.
g = cx.check(subject_id="bot")
if not g.allow:
    return refuse(g.reason)
```

Each SDK's README MUST show this example transliterated into the
language's idiomatic form.

---

## Appendix A — Subject and participant shapes

### A.1 Subject ID format

Subject IDs follow a canonical 4-part format:

```
subject:<division_id>:<subject_type>:<slug>
```

Where:

| Segment        | Required | Notes                                                |
|----------------|----------|------------------------------------------------------|
| `subject`      | yes      | Literal prefix                                       |
| `division_id`  | yes      | Division identifier, e.g. `div_abc`                  |
| `subject_type` | no       | Classifies the subject: `chat` \| `sensor` \| `lead` \| `ticket` \| `journey`. Omit for legacy 3-part form. |
| `slug`         | yes      | Unique-within-division slug, e.g. `bot`, `cust_456`  |

The server stores the `subject_type` on the subject record and uses it
to select the trace pattern (Appendix C) that governs trace grouping,
completeness conditions, and deviation rules.

On the wire, the server accepts any non-empty string. The SDK examples
in this spec use bare slugs (`"bot"`, `"cust"`). Internally the server
resolves them to the API key's division. Legacy formats (3-part
`user:ws_xxx:bot`, 4-part `subject:div_abc:chat:bot`) are also
accepted but not required.

Examples:

| Form       | Example                          | Use case                        |
|------------|----------------------------------|---------------------------------|
| Bare slug  | `bot`                            | Primary form — server resolves  |
| 4-part     | `subject:div_abc:chat:bot`       | Explicit division + type        |
| 3-part     | `user:ws_xxx:bot`                | Legacy — still accepted         |

The SDK helper `subjectTypeFromSubjectId()` / `subject_type_from_subject_id()`
extracts the type segment (or `null` / `None` when absent).

### A.2 Participant shape

```json
{
  "subject_id": "subject:div_abc:chat:bot",
  "role":       "agent",
  "kind":       "agent",
  "subject_type": "chat",
  "metadata":   {}
}
```

| Field          | Type   | Notes                                              |
|----------------|--------|----------------------------------------------------|
| `subject_id`   | string | Canonical subject id (4-part or 3-part)            |
| `role`         | string | Free-form: `agent`, `customer`, `observer`, …     |
| `kind`         | string | Substrate: `agent`, `human`, `sensor`, `service`, `other` |
| `subject_type` | string?| Classifier: `chat`, `sensor`, `lead`, `ticket`, `journey` |
| `metadata`     | object | Free-form metadata                                 |

`role` and `kind` default to `"other"` when omitted.
`subject_type` is optional; when present it overrides server-side
inference from the subject id.

## Appendix B — Wire compatibility commitments

For a spec MINOR bump, server changes that constitute a breaking change:

- Removing or renaming a request field.
- Removing or renaming a response field that was previously documented
  as always-present.
- Changing an enum value.
- Tightening a previously permissive validation.

Server changes that do NOT constitute a breaking change:

- Adding a new optional response field.
- Adding a new enum value (SDKs MUST tolerate unknown values by exposing
  the raw string).
- Adding a new endpoint.
- Loosening a validation.

---

## Appendix C — Trace & Notification patterns

This appendix describes server-side concepts that affect ingestion
behavior and notification delivery. SDKs do not expose methods to
manipulate these patterns directly (use the operator API or dashboard),
but SDK callers benefit from understanding them.

### C.1 Trace patterns

A **trace pattern** defines how frames are grouped into traces for a
given subject type. When `reasoning_mode` is `per_trace` (§1.7), frames
are deferred until the trace closes, then reasoned as a batch.

Each subject type has a default pattern:

| Type      | Grouping strategy   | Trace closes when…                                |
|-----------|---------------------|---------------------------------------------------|
| `chat`    | `session`           | No new frame arrives within the session timeout   |
| `sensor`  | `temporal_window`   | A fixed time window expires                       |
| `lead`    | `workflow_stage`    | The lead transitions to a new stage               |
| `ticket`  | `workflow_stage`    | The ticket transitions to a new stage             |
| `journey` | `workflow_stage`    | A terminal stage is reached                       |

Workspaces may override the default pattern for a subject type. Custom
patterns are configured through the operator dashboard (not via the SDK).

When a trace closes:

1. The assigned frames are locked from further ingestion.
2. `reason_over_trace()` runs one reasoning pass over every frame in the
   trace (the dedicated trace queue).
3. Deviations detected during reasoning trigger notification dispatch
   via notification patterns (§C.2).
4. Each frame in the trace becomes available via `await_outcome()`.

### C.2 Notification patterns

A **notification pattern** maps a triggering event (deviation detection,
stage stuck, SLA breach, churn risk) to delivery channels (in-app, SMS,
WhatsApp, webhook) with per-pattern deduplication, severity gating, and
template rendering.

Built-in patterns:

| Pattern name   | Trigger                              | Default channels       |
|----------------|--------------------------------------|------------------------|
| `deviation`    | Any deviation detected during        | in-app                 |
|                | trace-level reasoning                |                        |
| `stage_stuck`  | Subject remains in a stage past its  | in-app, webhook        |
|                | expected duration                    |                        |
| `sla_breach`   | SLA threshold exceeded for a         | in-app, SMS, webhook   |
|                | ticket or lead                       |                        |
| `churn_risk`   | Churn-risk score crosses threshold   | in-app, WhatsApp, SMS  |

Workspaces may install custom notification patterns with their own
trigger conditions, channel routing, and templates. Custom patterns
are configured through the operator dashboard.

Notification dispatch is **non-blocking**: patterns matching a detected
deviation fire asynchronously after trace reasoning completes.
