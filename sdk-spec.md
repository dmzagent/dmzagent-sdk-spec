# DMZAgent SDK Specification

**Version:** see [`VERSION`](./VERSION) — currently `0.10.0`
**Status:** pre-1.0 (MINOR bumps may include breaking wire changes)
**Last updated:** 2026-09-09

This document defines the public surface every DMZAgent SDK MUST
implement. Every language binding — Python, TypeScript, C#, Java — is a
translation of this surface. Same constructor shape, same methods (under
language-idiomatic naming), same return types, same error hierarchy,
same wire protocol.

A spec tag (`v0.10.0`) corresponds 1:1 to a release tag in every SDK
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

#### Test mode and live mode

Keys are issued in one of two modes, distinguished by the key material
itself:

| Prefix     | Mode | Data                                        |
|------------|------|---------------------------------------------|
| `ck_test_` | test | Written to test-mode storage; never billed  |
| `ck_`      | live | Production data                             |

The mode is a property of the key, not a request parameter — there is no
header or field that switches it, and an SDK MUST NOT offer one. To move
between modes, the caller changes the key.

Every ingestion response carries `livemode` (§2.1) reflecting the mode
the request was authenticated in. SDKs MUST surface it on `EmitResult`
and `CaptureResult` (§7.1, §7.2) rather than discarding it: it is the
only signal distinguishing test data from production data in a response,
and a caller holding the wrong key otherwise has no way to notice.

`Idempotency-Key` scope is namespaced per mode (§1.8), so the same key
value used in test and in live does not collide.

### 1.3 Content-Type

All request bodies are JSON-encoded. SDKs MUST set
`Content-Type: application/json` on every POST.

### 1.4 User-Agent

SDKs MUST set:

```
User-Agent: dmzagent-<language>/<spec-version>
```

Examples: `dmzagent-python/0.10.0`, `dmzagent-typescript/0.10.0`,
`dmzagent-csharp/0.10.0`, `dmzagent-java/0.10.0`. Optional suffix in the
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

### 1.8 Idempotency

`POST /v1/agent-stream/event` accepts a caller-generated
`Idempotency-Key` request header. It makes retrying a request safe: a
retry that carries the same key does not create a second interaction,
ingest a second frame, or trigger a second reasoning pass.

```
Idempotency-Key: <caller-generated unique string>
```

Server behaviour, which SDKs MUST NOT attempt to reimplement locally:

| Situation                                    | Response                                        |
|----------------------------------------------|-------------------------------------------------|
| Key unseen                                   | Request proceeds normally; response is stored   |
| Key seen, original completed                 | The stored response is replayed **verbatim**, including its original status code |
| Key seen, original still in flight           | `409` → `ConflictError` (§3)                    |

Scope is `(workspace, mode, key)`: the same key value is independent
across workspaces, and independent between test and live mode (§1.2).

An in-flight claim is held for 60 seconds. Past that the server presumes
the original request died mid-flight and lets a retry reclaim the key,
so a dropped connection does not strand a key.

Requirements on SDKs:

- The parameter MUST be optional and caller-supplied. An SDK MUST NOT
  generate a key on the caller's behalf — a key auto-generated per call
  is unique per call and therefore deduplicates nothing, while a key
  derived from payload content would silently collapse two legitimately
  distinct events that happen to be identical.
- A replayed response is indistinguishable from the original at the
  transport layer and MUST be returned as a normal result.
- `409` MUST surface as `ConflictError`, not as a retry the SDK performs
  itself. The caller decides whether to wait and retry.

---

## 2. Endpoints

### 2.1 POST /v1/agent-stream/event

Emit one event into the agent stream.

#### Request headers

| Header            | Required | Notes                                          |
|-------------------|----------|------------------------------------------------|
| `Authorization`   | yes      | `Bearer ck_…` (§1.2)                           |
| `Content-Type`    | yes      | `application/json` (§1.3)                      |
| `Idempotency-Key` | no       | Caller-generated; makes retries safe (§1.8)    |

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
  "follow_my_data": "/v1/frames/frame_xyz/story",
  "livemode": true
}
```

`livemode` is `true` when the request authenticated with a live key and
`false` for a test key (§1.2). It is always present on a `200`, and is
the only field in the response that distinguishes test data from
production data — SDKs MUST expose it rather than drop it.

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
  "route_latency_ms": 18.7,
  "pending_approval_id": null
}
```

`state` ∈ {`closed`, `half_open`, `open`}.
`allow` is `false` only when `state == open`.
`warning` is `true` when `state == half_open`.

`fired_policies` is an array of `{cb_policy_id, name, action}` objects.
`action` ∈ {`warn`, `open`, `require_approval`}.
`anchor` is an object `{ledger_index, hash}` or `null`.

`pending_approval_id` is the approval this check is waiting on, or
`null`. It is non-null only when a policy fired with action
`require_approval` and no human has decided yet — and in that case
`allow` is `false`, because an action awaiting approval has not been
approved.

**A denial that names an approval is a different denial.** A caller who
gets `allow=false` with no `pending_approval_id` has been refused;
a caller who gets one has been asked. That is the whole difference
between a breaker and a human-in-the-loop control, and it is one field
because the caller has to branch on it: refuse the user, or show them
the approval and wait (§2.8).

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

### 2.7 GET /v1/frames/{frame_id}/story

Narrative lineage for one frame: where it came from, what was received,
how it was read, what the canons matched, and what was concluded. This
is the endpoint `await_outcome()` (§5.10) polls, and the target of the
`follow_my_data` path returned by §2.1.

#### Path parameters

| Parameter  | Type   | Notes                                    |
|------------|--------|------------------------------------------|
| `frame_id` | string | as returned by `EmitResult.frame_id`     |

#### Query parameters

| Parameter      | Type   | Required | Notes                                         |
|----------------|--------|----------|-----------------------------------------------|
| `workspace_id` | string | no       | Narrow to one workspace's perspective. Omitted (the normal case), the whole division's traces are returned. |

Ingestion is **division-scoped**: the division is resolved from the
`subject_id` (Appendix A), the frame fans out to every workspace in that
division, and one reasoning trace is produced per workspace —
`n_workspaces` in §2.1 is exactly that count. The frame's natural scope is
therefore the division, and SDKs MUST NOT send `workspace_id` unless the
caller explicitly asked to narrow to a single perspective.

#### Response body

```json
{
  "frame_id":      "frame_xyz",
  "subject_id":    "subject:div:customer:acme",
  "division_id":   "div",
  "workspace_id":  null,
  "workspace_ids": ["ws_1", "ws_2"],
  "outcome":       "applied",
  "occurred_at":   "2026-08-30T12:00:00Z",
  "ingested_at":   "2026-08-30T12:00:01Z",
  "ingestion":     { },
  "frame":         { },
  "reasoning":     [ { "trace_id": "trace_1", "workspace_id": "ws_1", "outcome": "applied" } ],
  "soul_changes":  [ ],
  "ledger":        [ ],
  "summary": {
    "trace_count":     2,
    "workspace_count": 2,
    "complete":        true,
    "tags_fired":      3,
    "soul_versions":   [],
    "ledger_anchored": true
  }
}
```

Every entry in `reasoning[]` carries the `workspace_id` that produced it,
so the N perspectives are distinguishable. `division_id` names the owning
division; `workspace_ids` lists the workspaces the frame fanned out to;
`workspace_id` echoes the caller's filter and is `null` for a
division-scoped read.

#### Frame-level `outcome`

A frame has no intrinsic outcome — it has one per workspace. `outcome` is
a **fold** over `reasoning[]`, computed server-side so that four SDKs
cannot arrive at four different answers, with this precedence:

    failed  >  held  >  applied  >  no_change  >  skipped

Severity first, deliberately: a caller testing `outcome == "applied"` must
not be handed the more favourable of two perspectives when another
workspace's reasoning errored. `outcome` is `null` when no trace has been
recorded yet. The per-workspace value remains authoritative and is always
available in `reasoning[]`.

#### Completion

`summary.complete` is `true` when every workspace the frame fanned out to
has reported — that is, `trace_count >= workspace_count` with at least one
trace. This is the termination condition for `await_outcome()` (§5.10),
and it lines up with `n_workspaces` from the ingest ack (§2.1): that many
traces are expected, and fewer means the fan-out is still running.

Polling on anything else is wrong. A response that merely parses is not a
finished story: the traces arrive as each workspace completes.

---

### 2.8 GET /v1/approvals

List approvals awaiting a human decision.

An approval exists because a circuit-breaker policy fired with action
`require_approval` (§2.2). The action it guards has **not** run and will
not run until a human decides. The customer renders that decision in
their own product; this endpoint is what they render it from.

#### Query parameters

| Parameter    | Type    | Required | Notes                                                       |
|--------------|---------|----------|-------------------------------------------------------------|
| `status`     | enum    | no       | `pending` \| `approved` \| `declined` \| `expired`; default `pending` |
| `subject_id` | string  | no       | restrict to one subject                                      |
| `limit`      | integer | no       | 1–100, default 25                                            |
| `cursor`     | string  | no       | opaque; from a previous response's `next_cursor`             |

#### Response body

```json
{
  "approvals": [
    {
      "approval_id":   "apr_7f3c9a1b",
      "status":        "pending",
      "subject_id":    "user:ws_xxx:checkout-bot",
      "interaction_id": "ix_2b8e",
      "frame_id":      "fr_91ac",
      "action":        {"tool": "refund.issue", "args": {"amount": 9900}},
      "reason":        "refund above the reviewed ceiling",
      "fired_policies": [
        {"cb_policy_id": "cbp_11", "name": "refund ceiling", "action": "require_approval"}
      ],
      "requested_at":  "2026-09-09T12:00:00Z",
      "expires_at":    "2026-09-09T12:15:00Z",
      "on_expiry":     "decline",
      "anchor":        {"ledger_index": 40197, "hash": "b1c4…"},
      "decision":      null
    }
  ],
  "next_cursor": null
}
```

`next_cursor` is `null` on the last page.

#### The white-label contract

Everything needed to render the decision is in the approval object, and
**nothing in it is DMZAgent's presentation**. There is no message string
written for an end user, no logo, no colour, no copy. `reason` and
`fired_policies[].name` are the operator's own policy names — the words
the customer chose when they wrote the policy — so a customer's UI shows
their vocabulary, not ours.

An SDK MUST NOT synthesise display text from these fields. A field that
renders the same in every customer's product is a field DMZAgent has
branded, which is the thing this endpoint exists to avoid.

`action` is the call the agent was about to make, verbatim: the caller
already knows how to describe its own tools, and is the only party that
does.

---

### 2.9 POST /v1/approvals/{approval_id}/decision

Approve or decline a pending approval.

#### Path parameters

| Parameter     | Type   | Notes                                  |
|---------------|--------|----------------------------------------|
| `approval_id` | string | as returned by §2.8                     |

#### Request body

| Field         | Type   | Required | Notes                                                     |
|---------------|--------|----------|-----------------------------------------------------------|
| `decision`    | enum   | yes      | `approve` \| `decline`                                     |
| `actor_id`    | string | yes      | the deciding human, in the **customer's** namespace        |
| `reason`      | string | no       | free text, recorded on the ledger entry                    |
| `actor_label` | string | no       | display name for the customer's own audit view             |

#### Response body

The decided approval object (§2.8 shape) with `status` no longer
`pending` and `decision` populated:

```json
{
  "approval_id": "apr_7f3c9a1b",
  "status":      "approved",
  "decision": {
    "decision":    "approve",
    "actor_id":    "acct_4471",
    "actor_label": "Dana R.",
    "reason":      "verified the order by phone",
    "decided_at":  "2026-09-09T12:04:31Z"
  },
  "anchor": {"ledger_index": 40202, "hash": "9ee0…"}
}
```

#### A human decided, and the record says which one

`actor_id` is required, and an SDK MUST NOT default it, derive it from
the API key, or send a placeholder. The key identifies the *integration*;
the point of a human-in-the-loop control is that a *person* is on the
other end of it, and an approval whose actor is the integration that
requested it records nobody.

DMZAgent does not resolve `actor_id` against any directory. It is opaque,
it is the customer's own identifier, and it is stored and returned as
given — which is what keeps the flow white-label: the customer's users
never need accounts here.

#### Deciding twice

A second decision on an already-decided approval is `409` →
`ConflictError`. It is not an error to retry the *same* call — an
`Idempotency-Key` (§1.8) replays the original decision rather than
conflicting. Without one, two operators who click at the same moment
produce one decision and one conflict, and the conflict is the honest
answer: the approval was already settled, by someone else.

An approval past `expires_at` is `409` as well, with `status: "expired"`
in the body. The window closing is not a decision the second operator
can win.

#### Expiry fails closed

`on_expiry` is `decline` and an SDK MUST NOT offer a way to change it to
`approve`. An approval that becomes an allow because nobody looked at it
is not a human-in-the-loop control; it is a delay with extra steps.

---

### 2.10 GET /v1/incidents

The incident and remediation ledger.

Every breaker that opened, every approval that was decided, and every
remediation that ran is an entry. This is the append-only record behind
the `anchor` that §2.2, §2.8 and §2.9 return — the same
`{ledger_index, hash}` a caller was handed at the time, now readable.

#### Query parameters

| Parameter    | Type    | Required | Notes                                                        |
|--------------|---------|----------|--------------------------------------------------------------|
| `status`     | enum    | no       | `open` \| `remediated` \| `accepted` \| `all`; default `all`  |
| `subject_id` | string  | no       | restrict to one subject                                       |
| `since`      | string  | no       | ISO-8601; entries at or after this instant                    |
| `until`      | string  | no       | ISO-8601; entries strictly before this instant                |
| `limit`      | integer | no       | 1–100, default 25                                             |
| `cursor`     | string  | no       | opaque; from a previous response's `next_cursor`              |

#### Response body

```json
{
  "incidents": [
    {
      "incident_id":  "inc_5d2a70",
      "status":       "remediated",
      "kind":         "cb_open",
      "subject_id":   "user:ws_xxx:checkout-bot",
      "frame_id":     "fr_91ac",
      "opened_at":    "2026-09-09T11:58:02Z",
      "closed_at":    "2026-09-09T12:04:31Z",
      "reason":       "refund above the reviewed ceiling",
      "fired_policies": [
        {"cb_policy_id": "cbp_11", "name": "refund ceiling", "action": "require_approval"}
      ],
      "remediations": [
        {
          "remediation_id": "rem_88fe",
          "kind":           "approval",
          "approval_id":    "apr_7f3c9a1b",
          "outcome":        "approved",
          "actor_id":       "acct_4471",
          "reason":         "verified the order by phone",
          "occurred_at":    "2026-09-09T12:04:31Z",
          "anchor":         {"ledger_index": 40202, "hash": "9ee0…"}
        }
      ],
      "anchor": {"ledger_index": 40197, "hash": "b1c4…"}
    }
  ],
  "next_cursor": "eyJpIjo0MDE5N30"
}
```

`kind` ∈ {`cb_open`, `cb_half_open`, `policy_fired`, `approval_required`}.
`remediations[].kind` ∈ {`approval`, `policy_change`, `manual`, `auto`}.
`remediations` is ordered oldest first and MAY be empty — an incident
nobody has answered yet is an incident with no remediation, not an
absent incident.

#### Ordering

Entries are returned newest first, ordered by `ledger_index` descending —
never by `opened_at`. Two incidents opened in the same second have an
order, and it is the order the ledger recorded them in; sorting by a
timestamp that cannot separate them re-orders them by whatever the
tiebreak happens to be.

#### The ledger is append-only

There is no `PATCH`, no `DELETE`, and no endpoint that closes an
incident. A remediation is *appended*; the incident's `status` is a fold
over what has been appended to it. An SDK MUST NOT expose a method that
implies otherwise.

`anchor` on the incident is the entry that opened it. `anchor` on each
remediation is that remediation's own entry. A caller who recorded an
anchor at check time (§2.2) can find exactly that entry here and compare
hashes; an anchor that does not match the ledger is the one alarm this
endpoint exists to make possible.

---

## 3. Error handling

The server returns standard HTTP status codes. Every SDK MUST map them
to a typed exception hierarchy:

| Status | Canonical type      | Meaning                              |
|--------|---------------------|--------------------------------------|
| 400    | `ValidationError`   | malformed payload                    |
| 401    | `AuthError`         | API key missing / invalid / revoked  |
| 403    | `PermissionError`   | key valid but lacks scope            |
| 409    | `ConflictError`     | an `Idempotency-Key` request is already in flight (§1.8), or an approval is already decided or expired (§2.9) |
| 422    | `ValidationError`   | well-formed but unprocessable (bad event / rulebook) |
| 429    | `RateLimitError`    | rate cap reached — retry after `Retry-After` |
| 5xx    | `ServerError`       | transient — safe to retry            |
| other  | `DMZAgentError`    | unexpected status                    |

`CBOpenError` (canonical name) is NOT raised from the HTTP layer. It is
raised by the `guard()` context-manager / using-block when
`raiseOnOpen=true` and the check returns `allow=false`.

Every exception MUST expose:

- `message: string`
- `statusCode: number | null`
- `body: object | string | null`

`RateLimitError` additionally exposes:

- `retryAfter: number | null` — seconds until retrying can succeed,
  parsed from the response's `Retry-After` header (delta-seconds form);
  `null` when the header is absent or unparseable. SDKs MUST NOT sleep
  or retry automatically — surface the value and let the caller decide.

`ConflictError` carries no extra fields beyond the common three. It is
distinct from `ServerError` because it is **not** a transient fault: the
duplicate is the caller's own earlier request, still running. Retrying
the same `Idempotency-Key` after a short pause returns the original
response rather than a second side effect. SDKs MUST NOT retry it
automatically (§1.8).

The same type covers a settled approval (§2.9) for the same reason: the
call did not fail, it lost. Retrying cannot win, and an SDK that treats
it as transient turns a second operator's decline into a retry loop
against a decision that already stands. The body carries the approval's
current `status`, which is how a caller tells the two 409s apart.

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
| `cb_cache_ttl`     | duration | no | `0` (disabled)  |
| `cb_cache_max_entries` | integer | no | `1024`      |
| `cb_cache_on_error`| enum     | no | `raise`         |

The three `cb_cache_*` parameters configure the circuit-breaker state
cache; see §4.4. The cache is OFF unless `cb_cache_ttl` is greater than
zero.

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

### 4.4 Circuit-breaker state cache

`check()` is a network round trip on a path that callers put in front of
sensitive actions, so it is often the only synchronous DMZAgent call in a
request. An in-process cache removes that round trip for repeated checks
on the same subject.

**The cache is off by default and MUST stay off unless the caller sets
`cb_cache_ttl` above zero.** A caching safety check that nobody asked for
is a worse failure than a slow one.

#### What the caller is choosing

A cached `closed` is an allow the server might no longer give. **The TTL
is the maximum time a newly-opened breaker can go unobserved by this
client.** An SDK MUST state that in the documentation of every
`cb_cache_*` parameter it exposes, in those terms.

The cache applies the same TTL to every state. An SDK MUST NOT invent a
different lifetime per state: choosing to hold a deny longer than an
allow is a safety policy, and it belongs to the caller who set the TTL,
not to the SDK.

#### Behaviour

1. The cache key is `(scope, scope_ref)` — a `subject` entry and an
   `interaction` entry with the same id are distinct.
2. It is per client instance and in-process. An SDK MUST NOT share it
   across clients or write it anywhere outside the process.
3. Only a successful check populates it. Errors are never cached.
4. An entry older than `cb_cache_ttl` MUST NOT be served except under
   the `last_known` error policy below.
5. The cache is bounded at `cb_cache_max_entries`, evicting
   least-recently-used entries first. A client that checks many subjects
   MUST NOT grow without limit.
6. It MUST be safe to use from as many threads as the client itself is
   (§4.2).
7. `check(fresh = true)` MUST bypass any cached entry, perform the
   request, and replace the entry.

#### Every result says where it came from

A cached result MUST carry `cached = true` and `cache_age` — the age of
the entry when it was served. A fresh result MUST carry `cached = false`
and a zero age. A caller recording a denial has to be able to tell that
it read four-second-old state, and cannot if the SDK hides it.

`latency_ms`, `route_latency_ms`, `checked_at` and `raw` on a cached
result are the values from the check that actually happened. They
describe that check, and an SDK MUST NOT rewrite them to describe the
cache hit.

#### When the check fails

`cb_cache_on_error` decides what happens when the request fails —
network error, timeout, or 5xx:

| Value        | Behaviour                                                     |
|--------------|---------------------------------------------------------------|
| `raise`      | Propagate the error. This is the behaviour of an SDK with no cache, and the default. |
| `last_known` | Serve the last cached entry for that key even if it has expired, marked `cached = true` and `stale = true`. When there is no entry, propagate the error. |

`stale` marks every result served on that path, whether or not the entry
had expired: what the caller needs to know is that the server was asked
and could not answer, so this came from memory. An entry served normally
inside its TTL is `cached` and NOT `stale`.

`last_known` MUST NOT be reachable without `cb_cache_ttl` above zero:
there is nothing to fall back to until the caller has opted into the
cache.

A `stale` result is the client answering from memory while DMZAgent is
unreachable. It MUST be marked, and an SDK MUST NOT extend a normal TTL
to cover an outage silently.

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

### 5.6 `check(subject_id? | interaction_id?, fresh=false) → CheckResult`

Circuit-breaker check. Pass EXACTLY ONE of `subject_id` or
`interaction_id`. Both nil or both set MUST throw a validation error.

`fresh = true` bypasses the state cache (§4.4) and refreshes it. With the
cache disabled — the default — it has no effect.

### 5.7 `guard(subject_id? | interaction_id?, raise_on_open=false, fresh=false) → context-managed CheckResult`

Resource-scoped wrapper around `check()`; `fresh` is passed through to
it (§4.4). Language idioms:

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
`GET /v1/frames/{frame_id}/story` (§2.7) at a backoff interval (start
100ms, double to max 2s, cap at `timeout`). Raises `TimeoutError`
if the timeout is reached before reasoning completes — raised as
`ServerError` (§8.5) with `timed out` in the message. §8.5 carries no
timeout row and all four SDKs already raise their `ServerError`
equivalent; introducing a dedicated type would break every caller
currently catching it, so it is deferred to the next minor.

The poll MUST terminate on `summary.complete == true` and MUST NOT
terminate merely because a response parsed — the story endpoint answers
successfully throughout the fan-out, returning traces as each workspace
finishes, so returning the first non-erroring response yields a
half-finished story.

The poll MUST NOT send `workspace_id`. The frame is division-scoped
(§2.7); sending one narrows the result to a single perspective and makes
`complete` mean "that workspace finished" rather than "reasoning
finished".

This is the **poll variant** of the three retrieval modes (poll, webhook,
stream). Use it when the calling code needs a synchronous-feeling
response and can tolerate up to `timeout` latency.

Parameters:

- `frame_id: string` — returned by `capture().frame_id`
- `timeout?: float` — seconds, default 30.0. MUST cap at 120.0.

Returns `OutcomeResult` (§7.3) with per-workspace reasoning results.

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

### 5.16 `list_approvals(status?, subject_id?, limit?, cursor?) → ApprovalPage`

List approvals awaiting a human decision (§2.8). This is the read half
of the white-label human-in-the-loop control: the caller renders these
in their own product.

- `status: string` — OPTIONAL, default `pending`
- `subject_id: string` — OPTIONAL
- `limit: integer` — OPTIONAL, 1–100, default 25
- `cursor: string` — OPTIONAL

Returns `ApprovalPage` (§7.11).

An SDK MUST NOT auto-paginate inside this method. A caller who asked for
25 got 25, and a method that quietly walks every page turns one bounded
request into an unbounded one against a ledger that only grows. Offer
`iter_approvals()` (§5.17) for callers who want the walk, and make them
name it.

---

### 5.17 `iter_approvals(...) → iterator<Approval>`

Lazy iteration over §5.16, following `next_cursor` until it is `null`.
Same parameters minus `cursor`.

Each language exposes its idiomatic lazy sequence: a generator in
Python, an async iterable in TypeScript, `IAsyncEnumerable` in C#, a
`Stream` in Java. An SDK MUST fetch a page only when the consumer asks
for an item beyond the ones it holds — a method named for laziness that
buffers everything first is the auto-pagination §5.16 refuses, renamed.

---

### 5.18 `decide_approval(approval_id, decision, actor_id, reason?, actor_label?) → Approval`

Approve or decline (§2.9).

- `approval_id: string` — REQUIRED
- `decision: string` — REQUIRED, `approve` | `decline`
- `actor_id: string` — REQUIRED, the deciding human in the caller's own namespace
- `reason: string` — OPTIONAL
- `actor_label: string` — OPTIONAL

Returns the decided `Approval` (§7.12).

An SDK MUST reject an empty or missing `actor_id` locally, as a
`ValidationError`, without a round trip. The server rejects it too; the
reason to also refuse it here is that a caller who has not got a human's
identity at this point does not have a human, and the failure should
land where the mistake is.

The message MUST name the parameter **as that SDK spells it** — `actorId`
where the binding is camelCase — not the canonical wire key. The corpus
asserts the substring `actor` for exactly this reason: a vector that
demanded `actor_id` would be satisfiable only by the one binding that
spells it that way, and would have every other SDK name a parameter its
callers do not have.

Convenience wrappers `approve_approval(...)` / `decline_approval(...)`
MAY be offered. If they are, they take the same required `actor_id` and
MUST NOT be reachable without it.

---

### 5.19 `get_incidents(status?, subject_id?, since?, until?, limit?, cursor?) → IncidentPage`

Read the incident and remediation ledger (§2.10).

- `status: string` — OPTIONAL, default `all`
- `subject_id: string` — OPTIONAL
- `since: string` — OPTIONAL, ISO-8601
- `until: string` — OPTIONAL, ISO-8601
- `limit: integer` — OPTIONAL, 1–100, default 25
- `cursor: string` — OPTIONAL

Returns `IncidentPage` (§7.13). The same no-auto-pagination rule as
§5.16 applies.

---

### 5.20 `iter_incidents(...) → iterator<Incident>`

Lazy iteration over §5.19, on the terms of §5.17.

---

### 5.21 What the SDK does not offer

There is no `close_incident`, no `resolve_incident`, and no method that
edits a ledger entry, because §2.10 has no endpoint for one. An SDK MUST
NOT add a client-side convenience that reads as closing an incident —
appending an approval decision is how an incident reaches
`remediated`, and a method that says otherwise describes a ledger this
one is not.

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

> **Reserved numbers.** §7.4 and §7.7 are vacant. They held types that
> were removed before 0.7.0 and are left unused so the section numbers
> cited throughout the four SDK sources stay stable. Do not renumber the
> sections below to close the gaps; assign new types the next free
> number instead.

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
| `livemode`        | boolean?              | `true` for a live key, `false` for a test key (§1.2) |
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
| `livemode`        | boolean?              | `true` for a live key, `false` for a test key (§1.2) |
| `raw`             | object                | the full server JSON response                  |

### 7.3 `OutcomeResult`

Returned by `await_outcome()`.

| Field             | Type                  | Notes                                          |
|-------------------|-----------------------|------------------------------------------------|
| `frame_id`        | string                | the frame that was reasoned                     |
| `outcome`         | string?               | fold over `reasoning[]` — see §2.7. `skipped` \| `no_change` \| `applied` \| `failed` \| `held`. Null before any trace is recorded |
| `division_id`     | string?               | division that owns the subject                 |
| `workspace_ids`   | array<string>         | workspaces the frame fanned out to             |
| `complete`        | boolean               | every workspace has reported (`summary.complete`) |
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
| `outcome`         | string                | per-workspace outcome — `skipped` \| `no_change` \| `applied` \| `failed` \| `held` |
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
| `cached`           | boolean         | served from the state cache (§4.4)                |
| `cache_age`        | duration        | age of the cache entry when served; zero if fresh |
| `stale`            | boolean         | the check failed; this is the last known state    |
| `pending_approval_id` | string \| null | the approval this denial is waiting on (§2.2)  |
| `raw`              | object          | the full server JSON response                    |

`pending_approval_id` is non-null only alongside `allow = false`. A
caller branching on `allow` alone still behaves correctly — it refuses —
which is why this field was added rather than a new state: an SDK that
did not know about approvals must not start allowing what it used to
deny.

`cached`, `cache_age` and `stale` describe how the caller got this
result, and have no counterpart on the wire. A client with the cache
disabled always reports `false`, zero, `false`.

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

### 7.11 `ApprovalPage`

Returned by `list_approvals()`.

| Field         | Type             | Notes                                    |
|---------------|------------------|------------------------------------------|
| `approvals`   | array<Approval>  | this page, newest first                  |
| `next_cursor` | string \| null   | null on the last page                    |
| `raw`         | object           | the full server JSON response            |

### 7.12 `Approval`

An item of `ApprovalPage.approvals`, and the return of
`decide_approval()`.

| Field             | Type            | Notes                                                  |
|-------------------|-----------------|--------------------------------------------------------|
| `approval_id`     | string          |                                                        |
| `status`          | enum            | `pending` \| `approved` \| `declined` \| `expired`   |
| `subject_id`      | string          | the subject whose action is held                        |
| `interaction_id`  | string?         |                                                        |
| `frame_id`        | string?         |                                                        |
| `action`          | object          | `{tool, args}` — the held call, verbatim                |
| `reason`          | string          | the operator's own policy words (§2.8)                  |
| `fired_policies`  | array<object>   | `[{cb_policy_id, name, action}]`                       |
| `requested_at`    | string          | ISO-8601                                                |
| `expires_at`      | string          | ISO-8601                                                |
| `on_expiry`       | enum            | always `decline` (§2.9)                                 |
| `anchor`          | object \| null  | `{ledger_index, hash}`                                 |
| `decision`        | object \| null  | `{decision, actor_id, actor_label, reason, decided_at}` |
| `raw`             | object          | the full server JSON response                           |

`decision` is `null` while `status` is `pending` or `expired`.

An SDK MUST expose `expires_at` as the language's instant type where it
has one, and MUST NOT expose a "seconds remaining" derived at parse
time. A countdown computed when the object was built is wrong by however
long the caller held it, and a caller rendering an approval deadline is
exactly the caller who will hold it.

### 7.13 `IncidentPage`

Returned by `get_incidents()`.

| Field         | Type             | Notes                                    |
|---------------|------------------|------------------------------------------|
| `incidents`   | array<Incident>  | this page, newest `ledger_index` first   |
| `next_cursor` | string \| null   | null on the last page                    |
| `raw`         | object           | the full server JSON response            |

### 7.14 `Incident`

| Field            | Type                | Notes                                                       |
|------------------|---------------------|-------------------------------------------------------------|
| `incident_id`    | string              |                                                             |
| `status`         | enum                | `open` \| `remediated` \| `accepted`                       |
| `kind`           | enum                | `cb_open` \| `cb_half_open` \| `policy_fired` \| `approval_required` |
| `subject_id`     | string              |                                                             |
| `frame_id`       | string?             |                                                             |
| `opened_at`      | string              | ISO-8601                                                    |
| `closed_at`      | string?             | ISO-8601, null while open                                   |
| `reason`         | string              |                                                             |
| `fired_policies` | array<object>       | `[{cb_policy_id, name, action}]`                           |
| `remediations`   | array<Remediation>  | oldest first, MAY be empty                                  |
| `anchor`         | object \| null      | the entry that opened the incident                          |
| `raw`            | object              | the full server JSON response                               |

`Remediation` carries `remediation_id`, `kind`
(`approval` \| `policy_change` \| `manual` \| `auto`), `approval_id?`,
`outcome`, `actor_id?`, `reason?`, `occurred_at`, and its own `anchor`.

An incident with `remediations` empty and `status` `open` is the normal
shape of something nobody has answered yet. An SDK MUST NOT collapse it
to null, an empty result, or an error.

### 7.15 Immutability

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
| `ApprovalPage`       | `ApprovalPage`     | `ApprovalPage`     | `ApprovalPage` (record) | `ApprovalPage` (record) |
| `Approval`           | `Approval`         | `Approval`         | `Approval` (record) | `Approval` (record) |
| `IncidentPage`       | `IncidentPage`     | `IncidentPage`     | `IncidentPage` (record) | `IncidentPage` (record) |
| `Incident`           | `Incident`         | `Incident`         | `Incident` (record) | `Incident` (record) |
| `Remediation`        | `Remediation`      | `Remediation`      | `Remediation` (record) | `Remediation` (record) |
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
| `list_approvals`   | `list_approvals`   | `listApprovals`     | `ListApprovals`     | `listApprovals`     |
| `iter_approvals`   | `iter_approvals`   | `iterApprovals`     | `IterApprovals`     | `iterApprovals`     |
| `decide_approval`  | `decide_approval`  | `decideApproval`    | `DecideApproval`    | `decideApproval`    |
| `approve_approval` | `approve_approval` | `approveApproval`   | `ApproveApproval`   | `approveApproval`   |
| `decline_approval` | `decline_approval` | `declineApproval`   | `DeclineApproval`   | `declineApproval`   |
| `get_incidents`    | `get_incidents`    | `getIncidents`      | `GetIncidents`      | `getIncidents`      |
| `iter_incidents`   | `iter_incidents`   | `iterIncidents`     | `IterIncidents`     | `iterIncidents`     |

### 8.3 Constructor parameters

| Canonical    | Python       | TypeScript   | C#           | Java         |
|--------------|--------------|--------------|--------------|--------------|
| `api_key`    | `api_key`    | `apiKey`     | `apiKey`     | `apiKey`     |
| `base_url`   | `base_url`   | `baseUrl`    | `baseUrl`    | `baseUrl`    |
| `timeout`    | `timeout`    | `timeout`    | `timeout`    | `timeout`    |
| `user_agent` | `user_agent` | `userAgent`  | `userAgent`  | `userAgent`  |
| `subject_type` | `subject_type` | `subjectType` | `SubjectType` | `subjectType` |
| `cb_cache_ttl` | `cb_cache_ttl` | `cbCacheTtl` | `cbCacheTtl` | `cbCacheTtl` |
| `cb_cache_max_entries` | `cb_cache_max_entries` | `cbCacheMaxEntries` | `cbCacheMaxEntries` | `cbCacheMaxEntries` |
| `cb_cache_on_error` | `cb_cache_on_error` | `cbCacheOnError` | `cbCacheOnError` | `cbCacheOnError` |

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
| `livemode`           | `livemode`          | `livemode`            | `Livemode`           | `livemode`              |
| `retry_after`        | `retry_after`       | `retryAfter`          | `RetryAfter`         | `retryAfter`            |
| `fired_policies`     | `fired_policies`    | `firedPolicies`       | `FiredPolicies`      | `firedPolicies`         |
| `route_latency_ms`   | `route_latency_ms`  | `routeLatencyMs`      | `RouteLatencyMs`     | `routeLatencyMs`        |
| `cached`             | `cached`            | `cached`              | `Cached`             | `cached`                |
| `cache_age`          | `cache_age_ms`      | `cacheAgeMs`          | `CacheAge`           | `cacheAge`              |
| `stale`              | `stale`             | `stale`               | `Stale`              | `stale`                 |
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
| `approval_id`        | `approval_id`       | `approvalId`         | `ApprovalId`         | `approvalId`            |
| `pending_approval_id`| `pending_approval_id`| `pendingApprovalId` | `PendingApprovalId`  | `pendingApprovalId`     |
| `actor_id`           | `actor_id`          | `actorId`            | `ActorId`            | `actorId`               |
| `actor_label`        | `actor_label`       | `actorLabel`         | `ActorLabel`         | `actorLabel`            |
| `expires_at`         | `expires_at`        | `expiresAt`          | `ExpiresAt`          | `expiresAt`             |
| `on_expiry`          | `on_expiry`         | `onExpiry`           | `OnExpiry`           | `onExpiry`              |
| `decided_at`         | `decided_at`        | `decidedAt`          | `DecidedAt`          | `decidedAt`             |
| `incident_id`        | `incident_id`       | `incidentId`         | `IncidentId`         | `incidentId`            |
| `remediation_id`     | `remediation_id`    | `remediationId`      | `RemediationId`      | `remediationId`         |
| `remediations`       | `remediations`      | `remediations`       | `Remediations`       | `remediations`          |
| `opened_at`          | `opened_at`         | `openedAt`           | `OpenedAt`           | `openedAt`              |
| `closed_at`          | `closed_at`         | `closedAt`           | `ClosedAt`           | `closedAt`              |
| `occurred_at`        | `occurred_at`       | `occurredAt`         | `OccurredAt`         | `occurredAt`            |
| `next_cursor`        | `next_cursor`       | `nextCursor`         | `NextCursor`         | `nextCursor`            |

### 8.5 Exceptions

| Canonical            | Python              | TypeScript            | C#                                       | Java                                    |
|----------------------|---------------------|-----------------------|------------------------------------------|-----------------------------------------|
| `DMZAgentError`     | `DMZAgentError`    | `DMZAgentError`      | `DMZAgentException`                     | `DMZAgentException`                    |
| `AuthError`          | `AuthError`         | `AuthError`           | `DMZAgentAuthException`                 | `DMZAgentAuthException`                |
| `PermissionError`    | `PermissionError`   | `PermissionError`     | `DMZAgentPermissionException`           | `DMZAgentPermissionException`          |
| `ValidationError`    | `ValidationError`   | `ValidationError`     | `DMZAgentValidationException`           | `DMZAgentValidationException`          |
| `RateLimitError`     | `RateLimitError`    | `RateLimitError`      | `DMZAgentRateLimitException`            | `DMZAgentRateLimitException`           |
| `ConflictError`      | `ConflictError`     | `ConflictError`       | `DMZAgentConflictException`             | `DMZAgentConflictException`            |
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
| `approval.requested` | A policy fired `require_approval` and an action is held | `Approval` (§7.12) |
| `approval.decided`   | A human approved or declined an approval | `Approval` (§7.12) |
| `incident.opened`    | A ledger entry opened an incident | `Incident` (§7.14) |
| `incident.remediated`| A remediation was appended to an incident | `Incident` (§7.14) |

`approval.requested` is the push half of the white-label control: a
customer who does not want to poll §2.8 receives the same object here
and renders it the same way. The `source` for approval events is
`/v1/approvals`; for incident events, `/v1/incidents`.

**A missed webhook must not become an approval.** Delivery is
at-least-once and not guaranteed; the approval's `expires_at` runs
regardless, and expiry declines (§2.9). A customer who builds only on
the webhook and never reads §2.8 will hold actions that quietly expire,
which is safe but invisible. SDK documentation MUST say so where it
documents these events.

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

### 11.1 What the corpus does and does not cover

Stated plainly, because "conformance is green" is otherwise read as
"the surface is verified", and at 0.10.0 that is not what it means.

`golden-envelopes.json` exercises 5 of the 15 methods in §5 —
`subject_says`, `tool_call`, `tool_result`, `observation`, and `check` —
across 2 of the 7 endpoints in §2 (`/v1/agent-stream/event` and
`/v1/cb/check`).

`outcome-vectors.json` (added 0.8.1) exercises `await_outcome()`
against the story endpoint (§2.7): that no `workspace_id` is sent, that
the poll terminates on `summary.complete` rather than on the first
response that parses, that `reasoning[].workspace_id` survives parsing,
and that the server's `outcome` fold is reported verbatim rather than
recomputed.

No fixture covers `capture()`, `conversation()`, `guard()`, `close()`,
the notification-prefs pair (§5.12, §5.13), or the division-config pair
(§5.14, §5.15). The absence is load-bearing, and 0.8.1 is the proof: the
`await_outcome()` defect survived four SDK implementations precisely
because nothing executed that path, and every one of the four was broken
in a different way when a fixture finally did.

Nor does any fixture cover the state cache (§4.4). The corpus asserts on
request and response envelopes, and a cache hit is the ABSENCE of a
request — there is no envelope to golden. Each SDK carries its own
cache tests instead, which means cache behaviour is held to four
independent readings of §4.4 rather than to one shared corpus.

Treat a green `spec-conformance` check as evidence about serialization,
signature verification, and error mapping — not as evidence that a
method works end-to-end.

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

- Python: `pyproject.toml` → `[tool.dmzagent] spec-version = "0.10.0"`
- TypeScript: `package.json` → `"dmzagent": {"specVersion": "0.10.0"}`
- C#: `Directory.Build.props` → `<DMZAgentSpecVersion>0.10.0</DMZAgentSpecVersion>`
- Java: `pom.xml` → `<dmzagent.spec.version>0.10.0</dmzagent.spec.version>`

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
