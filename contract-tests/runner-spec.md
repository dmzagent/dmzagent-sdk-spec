# Runner specification

Every SDK repo ships a tiny test harness that exercises the JSON corpus
in this directory. This file specifies what that harness MUST do — the
language is free, but the behavior is fixed.

## Goals

A runner exists to prove three things, in this order:

1. **Serialization parity:** the SDK serializes wire bodies that match
   the canonical envelope byte-for-byte after JSON normalization.
2. **Verifier parity:** the SDK's HMAC verifier accepts and rejects the
   same vectors as every other SDK's verifier.
3. **Error mapping parity:** the SDK raises the language-canonical
   exception for every HTTP status code in the mapping.

## How to wire it

### Step 1: Pin the spec tag

The SDK repo MUST check out a specific tag of `dmzagent-sdk-spec` in
CI. The pinned tag MUST match the spec version recorded in the SDK's
language-native manifest (see §11.1 of `sdk-spec.md`).

A submodule or git-checkout-action with a `ref:` pin is preferred over
copying fixtures into the SDK repo — that way fixture drift surfaces
as a CI failure rather than silent staleness.

### Step 2: Implement a stub HTTP transport

The SDK MUST accept a transport at construction time (Python:
`transport=` kwarg on `httpx.Client`; TypeScript: optional `fetch`
override; C#: `HttpMessageHandler`; Java: OkHttp `Interceptor`).

The runner installs a stub transport that:

- Captures the outgoing request (method, path, headers, body).
- Returns a fixture-controlled response (status + body).

### Step 3: Drive each corpus

#### golden-envelopes.json

```
for each fixture in fixtures:
  construct SDK client with api_key="ck_test_xxxxxxxxxxxxxxxxxxxxx"
  call client.<method>(**args)
  assert captured.path == fixture.expected_path
  assert json_normalize(captured.body) == json_normalize(fixture.expected_body)

for each fixture in validation_failures:
  expect the SDK to raise an exception whose canonical type matches
    fixture.expected_exception (one of "ValidationError",
    "ValidationError_or_ArgumentError")
  assert fixture.expected_message_contains in str(exception)
```

`json_normalize` MUST: sort keys at every level, omit insignificant
whitespace, omit any trailing newline.

#### signature-vectors.json

```
for each fixture in fixtures:
  if fixture.header contains "<COMPUTE>":
    compute v1 = hex(hmac_sha256(fixture.secret, f"{t}.{payload}"))
    substitute into header
  elif fixture.header contains "<COMPUTE_WITH_OTHER>":
    compute v1 with fixture.header_signed_with
    substitute into header
  result = sdk.verify_webhook_signature(
    fixture.payload,
    fixture.header,
    fixture.secret,
    fixture.tolerance_seconds,
    now=fixture.now_unix    # if the SDK exposes a clock override; else mock time
  )
  assert result == fixture.valid
```

If the SDK's helper raises on bad input rather than returning `false`,
the runner MUST accept either behavior — but document which one the
language picks in its own README. The spec is intentionally permissive
here.

#### error-mapping.json

```
for each fixture in fixtures:
  install stub transport returning (fixture.status, fixture.body)
  if fixture.method == "guard_with_raise_on_open":
    try:
      with client.guard(subject_id=..., raise_on_open=True) as g:
        pass
    catch exc:
      assert canonical_type(exc) == fixture.expected_exception
      for k, v in fixture.expected_fields.items():
        assert getattr(exc, k) == v
  else:
    try:
      client.<method>(**args)
    catch exc:
      assert canonical_type(exc) == fixture.expected_exception
      assert exc.status_code == fixture.expected_status_code
```

`canonical_type` MUST map the language exception class back to the
canonical name listed in §8.5 of `sdk-spec.md`. Each runner implements
this map.

## Reporting

The runner MUST exit non-zero on any assertion failure and emit a
line-per-fixture summary in this format:

```
✓ golden_envelopes/subject_says_minimal
✗ golden_envelopes/tool_call_basic — body mismatch
  expected: {"kind":"tool_call",...}
  got:      {"kind":"tool_call",...}
```

This output is consumed by the spec-conformance GitHub Actions step;
the format is stable enough for regex extraction.

## CI integration

The SDK repo's `spec-conformance.yml` workflow MUST:

1. Check out the SDK repo at the release branch.
2. Check out the spec repo at the pinned tag from the manifest.
3. Build the SDK.
4. Run the runner.
5. Set the GitHub status check `spec-conformance/<spec-version>` to
   success or failure.

The promote workflow in `dmzagent-sdk-spec/.github/workflows/promote.yml`
verifies this status before dispatching `publish.yml`.
