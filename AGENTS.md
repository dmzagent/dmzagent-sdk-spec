# AGENTS.md — briefing for SDK implementation agents

If you are an agent tasked with implementing a DMZAgent SDK in
TypeScript, C#, or Java, this file tells you what to read, what to
build, and what "done" means.

## Read first, in order

1. [`sdk-spec.md`](./sdk-spec.md) — the surface you are implementing.
   This is normative. If your language has an idiomatic deviation from
   snake_case naming, refer to §8 of the spec for the official map.
2. [`schemas/`](./schemas/) — JSON Schema definitions for every wire
   payload. Your serializer must produce output that validates against
   these. Your deserializer must accept any payload these validate.
3. [`contract-tests/golden-envelopes.json`](./contract-tests/golden-envelopes.json)
   — fixed inputs → expected wire bodies. Your SDK's serialization
   tests must round-trip these byte-for-byte.
4. [`contract-tests/signature-vectors.json`](./contract-tests/signature-vectors.json)
   — HMAC verification vectors for the webhook signature helper.
5. [`contract-tests/error-mapping.json`](./contract-tests/error-mapping.json)
   — HTTP status → exception type your SDK must raise.
6. [`contract-tests/runner-spec.md`](./contract-tests/runner-spec.md)
   — how to wire the language-specific contract test harness.

## What "done" means

Your SDK is done when:

- [ ] The repo's `spec-conformance.yml` CI workflow passes against this
      spec's tag.
- [ ] Every method in §6 of the spec is implemented under the language's
      naming convention (see §8 naming map).
- [ ] The exception hierarchy in §4 is mirrored exactly.
- [ ] The webhook signature verifier in §10 passes all vectors.
- [ ] The package is publishable to its registry (PyPI / npm / NuGet /
      Maven Central) with no provenance gaps.
- [ ] The README in the SDK repo shows the Quick-Start example from the
      spec, transliterated into the language.

## What you may NOT change

- **Wire protocol** — request and response shapes are fixed by the
  spec. If you discover a real bug, open an issue here, not a unilateral
  workaround in your SDK.
- **Method names from the naming map** — the canonical snake_case names
  in §8 dictate your idiomatic equivalents. No creative liberty.
- **Error mapping** — HTTP status → exception is fixed. Catch must be
  predictable across languages.
- **Version** — your SDK's published version must match this repo's
  `VERSION` exactly. No drift.

## What you SHOULD design idiomatically

- HTTP client choice (httpx / fetch / HttpClient / OkHttp).
- Threading and async model — match the language's expectations.
- Resource cleanup — `with`/`using`/`try-with-resources`/`close()`.
- Logging — use the language's standard logging facade.
- Build tooling — `pyproject.toml`/`package.json`/`.csproj`/`pom.xml`,
  pick the tool the ecosystem prefers.

## Open scope questions: don't decide unilaterally

If you encounter ambiguity in the spec, file an issue in `dmzagent-sdk-spec`
with the title `[ambiguity] §<section>: <one-line summary>`. Do not
guess and ship — language SDKs that drift in interpretation are exactly
what this spec exists to prevent.
