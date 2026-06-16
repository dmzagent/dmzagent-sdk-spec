# Changelog

All notable changes to the DMZAgent SDK specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning per `sdk-spec.md` §11.

## [Unreleased]

## [0.6.0] — 2026-06-13

### Added
- `/v1/notification-prefs` endpoint (GET/PUT) for email cadence, push, SMS,
  and WhatsApp notification preferences (§2.3, §2.4).
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
