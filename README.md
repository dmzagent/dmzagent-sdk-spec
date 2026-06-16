# dmzagent-sdk-spec

**The canonical wire contract for every DMZAgent SDK.**

This repository is the source of truth for what a DMZAgent SDK is. Every
language binding — Python, TypeScript, C#, Java — implements the same
surface defined here, gated by the same contract test corpus, released
under the same semantic version.

There is no runtime code in this repo. It contains:

- [`sdk-spec.md`](./sdk-spec.md) — the public surface every SDK must expose,
  in language-agnostic form, with a naming map from the canonical
  snake_case to each language's idiomatic conventions.
- [`schemas/`](./schemas/) — JSON Schema definitions for every request and
  response on the wire.
- [`contract-tests/`](./contract-tests/) — language-agnostic test corpus:
  golden envelopes, HMAC signature vectors, HTTP error mappings.
- [`VERSION`](./VERSION) — the current spec version. Every SDK release tag
  matches this exactly.
- [`.github/workflows/promote.yml`](./.github/workflows/promote.yml) — the
  release coordination workflow that publishes all SDKs in lockstep.

## How releases work

1. A change lands here (new field, new endpoint, new error code).
2. `VERSION` is bumped.
3. A spec tag `v<version>` is pushed.
4. Each SDK repo (`dmzagent-sdk-python`, `dmzagent-sdk-typescript`,
   `dmzagent-sdk-csharp`, `dmzagent-sdk-java`) opens a PR that pins to
   the new spec tag and implements any new surface.
5. Each SDK repo's `spec-conformance.yml` workflow runs the contract test
   corpus from `contract-tests/` against its build. Red = blocks the
   release.
6. When all four SDKs report green against the same spec tag, the
   `promote.yml` workflow here dispatches `publish` to each SDK repo in
   parallel.
7. The spec release is marked `promoted` only after all four registries
   confirm the package is live. A partial publish triggers a coordinated
   yank.

This is what makes the parity claim auditable. No SDK can ship a version
the spec hasn't blessed; no spec version is "released" until every SDK
implements it.

## Sister repos

| Repo                          | Language    | Registry         |
|-------------------------------|-------------|------------------|
| dmzagent-sdk-python          | Python      | PyPI             |
| dmzagent-sdk-typescript      | TypeScript  | npm              |
| dmzagent-sdk-csharp          | C#          | NuGet            |
| dmzagent-sdk-java            | Java        | Maven Central    |

## Versioning

Pre-1.0: MINOR bumps may include breaking wire changes; PATCH bumps must
remain backwards-compatible.

Post-1.0: MAJOR bumps for breaking changes only, with a one-MINOR
deprecation cycle.

## License

Apache-2.0 (same as the SDK repos).

## Maintainer

matt@eastern-shore-solutions.com
