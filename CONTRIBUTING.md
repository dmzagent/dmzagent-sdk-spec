# Contributing to dmzagent-sdk-spec

This repository is the source of truth for the DMZAgent SDK surface. A
change here causes a cascade through every language SDK — please treat
edits accordingly.

## Editing the spec

1. Discuss first. Open an issue describing the proposed change with the
   following template:
   - **What:** the surface or wire change in one sentence.
   - **Why:** the customer or operator need being met.
   - **Compat:** is this PATCH, MINOR, or MAJOR? (See `sdk-spec.md` §11.)
   - **Server PR:** link to the server-side implementation, if any.
2. PR the edit to `sdk-spec.md`. Update `VERSION` only when the change
   is ready to release.
3. Add or update entries in `contract-tests/` so every SDK can prove
   conformance.
4. If the change is MINOR or MAJOR, list it in `CHANGELOG.md` with the
   deprecation cycle (if any).

## Release workflow

A coordinated release follows these steps:

1. Merge all spec edits for the upcoming version.
2. Bump `VERSION`. Push tag `v<version>` to this repo.
3. Each SDK repo opens a `release/v<version>` branch that pins to the
   new spec tag.
4. Each SDK repo's `spec-conformance.yml` workflow runs against the new
   spec tag. Must be green.
5. Trigger `promote.yml` here. It dispatches `publish.yml` to all four
   SDK repos in parallel.
6. Once all four registries confirm the artifact is live, the spec
   release is marked `promoted`.
7. A partial publish triggers `yank.yml` to pull artifacts that already
   landed.

If any SDK can't pass conformance against the new spec, the release
blocks — fix the SDK or revert the spec. We do not ship partial
parity.

## What lives here vs in the SDK repos

| In `dmzagent-sdk-spec`              | In each SDK repo                       |
|--------------------------------------|----------------------------------------|
| The surface contract (`sdk-spec.md`) | The implementation                     |
| JSON schemas                         | The wire serializer / deserializer     |
| HMAC vectors                         | The signature verifier helper          |
| Error mappings                       | The exception classes                  |
| Release coordination workflow        | `spec-conformance.yml` + `publish.yml` |
| `VERSION` (source of truth)          | Pinned spec version in manifest        |

If you're tempted to put runtime code here, you're in the wrong repo.
If you're tempted to put a wire-level decision in the SDK repo, you're
in the wrong repo.
