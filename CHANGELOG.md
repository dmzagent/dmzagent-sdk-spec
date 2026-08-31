# Changelog

All notable changes to the DMZAgent SDK specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning per `sdk-spec.md` §11.

## [Unreleased]

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
- No wire change: `openapi.json` is untouched, and a server that has
  never heard of the cache serves a cached client identically. Every
  addition here is client-side.
- Unlike the four behaviours 0.8.0 documented — test/live mode,
  `livemode`, idempotency, `409 → ConflictError` — this one is
  implemented in all four SDKs in the same change that specifies it, so
  it does not add to the set of specified-but-unimplemented surface.

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
