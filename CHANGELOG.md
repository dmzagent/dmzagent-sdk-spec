# Changelog

All notable changes to the DMZAgent SDK specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning per `sdk-spec.md` §11.

## [Unreleased]

## [0.10.0] — 2026-09-09

### Added
- **A white-label human-in-the-loop control (§2.8, §2.9, §5.16–§5.18).**
  A circuit-breaker policy can now fire with action `require_approval`,
  which holds the action rather than refusing it, and the customer
  renders the decision inside their own product.

  The white-label part is a constraint on what these payloads may
  contain, not a theme: **no field carries DMZAgent presentation.**
  There is no message written for an end user, no logo, no copy.
  `reason` and `fired_policies[].name` are the operator's own policy
  words, and `action` is the held call verbatim, because the caller
  named its own tools and is the only party that can describe them. An
  SDK MUST NOT synthesise display text from these fields — a field that
  renders the same in every customer's product is a field we branded,
  which is the thing this endpoint exists to avoid.

  `actor_id` is REQUIRED on a decision and is the customer's own
  identifier for the deciding human. DMZAgent resolves it against no
  directory, which is what lets the customer's users decide without
  ever holding an account here. SDKs reject an empty one locally: a
  caller who has not got a human's identity at this point does not have
  a human, and the failure should land where the mistake is.

- **`pending_approval_id` on the check response (§2.2, §7.5).** The one
  field that tells a caller which kind of `allow = false` they were
  handed: a refusal, or an ask. It is additive on purpose — an SDK that
  does not know about approvals still reads `allow` and still refuses,
  because the alternative is an older client that starts allowing what
  it used to deny.

- **The incident and remediation ledger is readable (§2.10, §5.19,
  §5.20).** `anchor: {ledger_index, hash}` has been returned since
  0.5.0 and pointed into a ledger no SDK could open. `GET /v1/incidents`
  opens it: every breaker that opened, every approval decided, every
  remediation that ran — and a caller who recorded an anchor at check
  time can now find exactly that entry and compare hashes. An anchor
  that does not match is the one alarm this endpoint exists to make
  possible.

  It is **append-only**: no `PATCH`, no `DELETE`, no endpoint that
  closes an incident. A remediation is appended and `status` is a fold
  over what has been appended, so §5.21 forbids an SDK from offering a
  `close_incident` convenience that describes a ledger this is not.

  Ordered by `ledger_index` descending, never by `opened_at` — two
  incidents opened in the same second have an order, and it is the one
  the ledger recorded, not whatever a timestamp tiebreak produces.

- **`approval.requested`, `approval.decided`, `incident.opened`,
  `incident.remediated` webhooks (§9.2).** The push half of the same
  control. Documented with the failure they invite: delivery is
  at-least-once and not guaranteed, `expires_at` runs regardless, and a
  customer who builds only on the webhook holds actions that quietly
  expire.

### Changed
- **`ConflictError` also covers a settled approval (§3).** Same type,
  same reasoning: the call did not fail, it lost. Retrying cannot win,
  and an SDK that treats it as transient turns a second operator's
  decline into a retry loop against a decision that already stands. The
  body carries the approval's current `status`, which is how a caller
  tells the two 409s apart.
- **`fired_policies[].action` has an enum** — `warn`, `open`,
  `require_approval` — where it was previously an open string.

### Notes
- **Expiry fails closed and is not configurable.** `on_expiry` is
  `decline`, and SDKs MUST NOT offer a way to make it `approve`. An
  approval that becomes an allow because nobody looked at it is not a
  human-in-the-loop control; it is a delay with extra steps.
- **No auto-pagination (§5.16).** A caller who asked for 25 got 25.
  `iter_approvals` / `iter_incidents` exist for the walk and have to be
  named, because a method that quietly follows every cursor turns one
  bounded request into an unbounded one against a record that only
  grows.
- Contract vectors for the new surface are in
  `contract-tests/golden-envelopes.json` and `error-mapping.json`.
  Python implements 0.10.0 first; TypeScript, C# and Java stay pinned
  at 0.9.0 until they follow, and `promote.yml` gates the coordinated
  release on all four being at the same version.


## [0.9.0] — 2026-08-31

### Added
- **Circuit-breaker state cache (§4.4).** `check()` is a network round
  trip in front of a sensitive action, and is often the only synchronous
  DMZAgent call in a request. An opt-in per-client cache removes that
  round trip for repeated checks on the same subject.

  The cache is **off unless `cb_cache_ttl` is set above zero**, applies
  one TTL to every state, is bounded and LRU-evicted at
  `cb_cache_max_entries`, and never leaves the process. `check(fresh)`
  bypasses it.

  What the caller is choosing is written into the section in those
  terms: a cached `closed` is an allow the server might no longer give,
  and the TTL is the maximum time a newly-opened breaker can go
  unobserved by that client. An SDK MUST NOT hold a deny longer than an
  allow — the asymmetry is a safety policy and belongs to whoever set
  the TTL.
- **`cached`, `cache_age`, `stale` on `CheckResult` (§7.5).** How the
  caller got this result, with no counterpart on the wire. A caller
  recording a denial has to be able to tell it read four-second-old
  state. A cached result keeps the server's own `latency_ms`,
  `route_latency_ms`, `checked_at` and `raw` — those describe the check
  that happened and are not rewritten to describe the cache hit.
- **`cb_cache_on_error` (§4.4).** `raise` (default) is the behaviour of
  an SDK with no cache. `last_known` serves the last entry for that key
  even if expired, marked `stale`, when the check itself fails —
  answering from memory while DMZAgent is unreachable, which must be
  marked rather than folded into a normal cache hit.
- Naming map entries for all three constructor parameters (§8.3) and all
  three result fields (§8.4).

### Notes
- No wire change: the endpoint surface in `openapi.json` is unchanged —
  only its `info.version` moves with `VERSION`, alongside the three
  contract-test corpus files — and a server that has never heard of the
  cache serves a cached client identically. Every addition here is
  client-side.
- Specified and implemented together: the four SDK pull requests carrying
  §4.4 open alongside this one, so nothing here joins the
  specified-but-unimplemented surface even briefly. The SDKs cannot go
  green until this merges, because each now asserts its pin against this
  repository's `VERSION`.

## [0.8.1] — 2026-08-30

Closes the `await_outcome()` defect recorded as known at 0.8.0. The method
was unusable in all four SDKs; it now has a specified, division-scoped
contract and a termination condition.

### Fixed
- **`workspace_id` on `GET /v1/frames/{frame_id}/story` (§2.7) is now an
  optional filter, not a required selector.** Ingestion is division-scoped:
  the division is resolved from the subject, the frame fans out to every
  workspace in it, and one trace is produced per workspace. Requiring one
  workspace asked the caller to pick 1 of N perspectives — the wrong axis.
  No SDK sent it, so every `await_outcome()` call took a 422. Omitted, the
  whole division's traces are returned; supplied, the result narrows, so
  existing callers are unaffected.
- **`reasoning[].workspace_id` is now returned.** §7.3 has always required
  it. The traces table has always carried the column; the query did not
  select it, so callers received N traces with no way to tell the
  perspectives apart.
- **Ledger anchors resolve for canonical subject ids.** The walk derived
  its workspace by string-splitting the subject and taking part 1 only when
  it began with `user:`, so every `subject:<div>:<slug>` id resolved to
  none, the walk was skipped, and `summary.ledger_anchored` reported false
  for frames that were in fact anchored. Audit-log lookup had the same
  legacy-only assumption and now gathers across the division.

### Added
- **Frame-level `outcome` (§2.7, §7.3)** — a fold over `reasoning[]` with
  precedence `failed > held > applied > no_change > skipped`. Severity
  first, so a caller testing for `applied` is not handed the more
  favourable of two perspectives when another workspace errored. Computed
  server-side because four SDKs folding independently is precisely the
  drift this repository exists to prevent.
- **`summary.complete` and `summary.workspace_count` (§2.7)** — the
  termination condition for `await_outcome()`, tied to `n_workspaces` from
  the ingest ack (§2.1). Previously there was nothing to poll on: Python
  keyed on a top-level `outcome` that did not exist and always timed out,
  while TypeScript, Java and C# returned the first response that parsed —
  a half-finished story, since the endpoint answers successfully all the
  way through the fan-out.
- **`division_id` and `workspace_ids` (§2.7, §7.3)** — the scope the story
  was rendered over.
- **`held` added to the outcome enum (§7.3).** The server has emitted it
  since ST-8; the spec listed four of the five values.
- **`contract-tests/outcome-vectors.json`** — first fixtures covering
  `await_outcome()`. §11.1 recorded that no fixture covered the method,
  which is why the defect survived four implementations.

### Notes
- `workspace_id` remains accepted, so this is a patch release: no
  request that worked at 0.8.0 changes meaning.

## [0.8.0] — 2026-08-30

An audit of the specified surface against the server it describes and the
four SDKs that implement it. Most of this release is documenting behaviour
that already shipped but was never written down.

### Added
- **Test mode / live mode (§1.2).** Keys are issued as `ck_test_…` or
  `ck_…`; the mode is a property of the key, not a request parameter.
  Previously undocumented despite being fully implemented server-side.
- **`livemode` on ingestion responses (§2.1, §7.1, §7.2).** The server has
  always returned it; no SDK exposed it. It is the only field
  distinguishing test data from production data in a response.
- **Idempotency (§1.8).** `Idempotency-Key` on
  `POST /v1/agent-stream/event`, with replay-verbatim semantics, a
  `(workspace, mode, key)` scope, and a 60-second in-flight window. The
  mechanism existed server-side and was invisible to every SDK, so no
  caller could retry safely.
- **`409 → ConflictError` (§3).** A concurrent duplicate previously
  collapsed into generic `DMZAgentError`, leaving callers unable to
  distinguish "retry shortly" from an unexpected failure. Naming map
  entries added for all four languages (§8.5), and fixture
  `409_idempotency_conflict` added to `error-mapping.json`.
- **`GET /v1/frames/{frame_id}/story` (§2.7).** The endpoint
  `await_outcome()` polls and `follow_my_data` points at. It was absent
  from both §2 and `openapi.json`, which meant the spec could not be
  implemented from the spec alone.
- **§11.1 — what the contract corpus does and does not cover.** The corpus
  exercises 5 of 15 methods across 2 of 7 endpoints. Recorded because a
  green `spec-conformance` check was reasonably being read as broader
  assurance than it provides.

### Fixed
- `OutcomeResult` cross-reference in §5.10 pointed at §7.4; the type is
  at §7.3.
- Document header read `currently 0.6.0` and `Last updated 2026-06-13`
  after the 0.7.0 release bumped `VERSION` and `CHANGELOG` but not the
  document. §1.4's User-Agent examples were stale for the same reason.
- The 0.6.0 entry below named the notification endpoint
  `/v1/notification-prefs`. That path has never existed — the endpoint is
  `/v1/settings/notifications`, as §2.3/§2.4, `openapi.json`, and the
  server all state. Corrected in place; the spec body was always right.

### Known defects
- **`await_outcome()` (§5.10) cannot succeed against the current server**,
  for two independent reasons recorded in §2.7: the story endpoint
  requires a `workspace_id` the SDK has no way to obtain, and
  `OutcomeResult`'s top-level `outcome` discriminator does not exist in
  the response. Not fixed here — resolving it is an API-shape decision,
  not an editorial one. No contract fixture covers the method, which is
  why it survived four implementations.

### Notes
- §7.4 and §7.7 are reserved-vacant. The SDK sources cite section numbers
  extensively, so the gaps are held rather than closed by renumbering.

## [0.7.0] — 2026-07-10

### Added
- Error taxonomy (§3): `422 → ValidationError` (well-formed but
  unprocessable — bad event / rulebook) and `429 → RateLimitError`, a new
  canonical exception exposing `retryAfter` parsed from the `Retry-After`
  header (`DMZAgentRateLimitException` in C#/Java). Previously both
  statuses collapsed into the generic `DMZAgentError`.
- `contract-tests/error-mapping.json`: fixtures `422_unprocessable_validation`,
  `429_rate_limited_retry_after` (with a `headers` field runners MUST pass
  through to the stubbed response, and an `expected_retry_after` assertion),
  and `429_rate_limited_no_header` (`retryAfter` is `null`, never a guess).

## [0.6.0] — 2026-06-13

### Added
- `/v1/settings/notifications` endpoint (GET/PUT) for email cadence, push,
  SMS, and WhatsApp notification preferences (§2.3, §2.4). *(Corrected in
  0.8.0: this entry originally read `/v1/notification-prefs`, a path that
  never existed.)*
- `/v1/divisions/{id}/config` endpoint (GET/PUT) for per-division JSON
  configuration including `reasoning_mode` (§2.5, §2.6).
- `get_notification_prefs()` / `update_notification_prefs()` on the client
  (`NotificationPrefs` result type) (§5.12, §5.13, §7.8).
- `get_division_config()` / `update_division_config()` on the client
  (`DivisionConfig` result type) (§5.14, §5.15, §7.9).
- `reasoning_mode` concept documented in the ingestion contract (§1.7):
  `per_frame` (default — reason every frame) or `per_trace` (defer until
  trace closes, then batch-reason over all frames in the trace).
- 4-part subject ID format `subject:<div>:<type>:<slug>` documented in
  Appendix A, with optional `type` segment.
- `subject_type` field on participant shapes.
- Appendix C — Trace & Notification patterns overview.
- JSON Schemas for notification-prefs request/response and division-config
  request/response in `schemas/`.

### Changed
- Spec version bumped from `0.5.0` to `0.6.0`.
- `capture()` parameter `subject_id` now accepts the 4-part canonical form
  (backwards-compatible with 3-part).
- Naming maps updated for all new methods, result types, and fields (§8).

## [0.5.0] — 2026-05-30

First spec-gated release. Establishes the four-SDK lockstep contract.

### Added
- Canonical surface defined in `sdk-spec.md` (§§1–11).
- JSON Schemas for the two endpoints currently in scope:
  `/v1/agent-stream/event` and `/v1/cb/check`.
- Contract test corpus: `golden-envelopes.json`, `signature-vectors.json`,
  `error-mapping.json`.
- Webhook signature verification helper specification (§9).
- Naming map for Python / TypeScript / C# / Java (§8).
- Coordinated release workflow (`promote.yml`).

### Changed
- Python SDK supersedes its unilateral `0.1.0` line; new starting point
  is the lockstep `0.5.0` across all four implementations.

### Migration notes (for the existing Python SDK consumers)
- `EmitResult` gains optional fields (`frame_id`, `subject_id`,
  `outcome`, `triage_decision`, `tags_fired`, `scored_by_canons`,
  `soul_version`, `ledger_index`, `follow_my_data`) populated in
  synchronous mode. Async mode (`X-DMZAgent-Async: true`) preserves
  the previous minimal envelope.
- No breaking changes to method signatures.
