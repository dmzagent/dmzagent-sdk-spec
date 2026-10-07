# Contract tests

This corpus is what every SDK MUST pass to be considered conformant
with the spec version in [`../VERSION`](../VERSION).

The corpus is intentionally language-agnostic — JSON data files only,
no executable code. Each SDK ships its own thin runner that loads these
files and drives the SDK under test.

## Files

| File                       | Tests                                                  |
|----------------------------|--------------------------------------------------------|
| `golden-envelopes.json`    | Serialization round-trips for every event kind         |
| `signature-vectors.json`   | HMAC webhook verifier vectors                          |
| `error-mapping.json`       | HTTP status → exception type mapping                   |
| `step-vectors.json`        | Agent mode: reading a step's directive, unknown values fail closed |
| `runner-spec.md`           | How each SDK's runner must be wired                    |

## How a runner works

Each SDK repo contains a small test harness that:

1. Loads the JSON fixtures from this directory (pinned to a spec tag).
2. Drives the SDK with a stub HTTP transport.
3. Asserts the behavior described in each fixture.

Concretely:

- **Golden envelopes:** given a fixture's `args` (method + kwargs), the
  SDK MUST POST a body whose canonical JSON equals `expected_body`
  byte-for-byte (after sort_keys + no trailing whitespace
  normalization).
- **Signature vectors:** given `(payload, header, secret, tolerance)`,
  the SDK's `verify_webhook_signature` helper MUST return the expected
  `valid` boolean.
- **Error mappings:** given a fake transport returning the fixture's
  `status` and `body`, calling the SDK method MUST raise the exception
  whose canonical name matches `expected_exception`.

## Adding a vector

1. PR the new fixture entry here.
2. Update every SDK's runner (in their respective repos) so it covers
   the new entry.
3. Bump `VERSION` if the fixture represents new surface (MINOR) rather
   than a bug-fix vector for existing surface (PATCH).
