# Event fixtures (Flow A)

Hand-authored canonical events. CI validates each fixture against [`schemas/event.json`](../../schemas/event.json) on every PR.

## Inventory

| File | `action.type` | `policy.decision` | Conformance demonstrated |
|---|---|---|---|
| [`01-tool-call-allowed.json`](01-tool-call-allowed.json) | `tool_call` | `allowed` | L2 — explicit allow with rule_id and policy_hash |
| [`02-tool-call-blocked-external-email.json`](02-tool-call-blocked-external-email.json) | `tool_call` | `blocked` | L2 — block with rationale; matches §9 worked example |
| [`03-model-response-logged.json`](03-model-response-logged.json) | `model_response` | `logged_only` | L2 — passive logging path |
| [`04-session-open-l1-only.json`](04-session-open-l1-only.json) | `session_open` | *(none)* | L1 — minimal required fields, no policy block |

## Signature placeholders

The `signature.value` fields are 88-character base64 placeholders that pass schema validation but **do not cryptographically verify**. They exist so the JSON files are well-formed for schema CI without anyone needing to commit a private key.

The Python SDK's `tests/test_roundtrip.py` re-signs each fixture with a deterministic test keypair and asserts that verification succeeds — that's where signature semantics are exercised. Phase 0 fixtures intentionally split schema-conformance (here) from signature-correctness (in the SDK).

## Adding new fixtures

1. Author a `NN-short-name.json` file — number sequentially.
2. Add a row to the table above.
3. Confirm the fixture passes schema validation: `python -m openagp.tools.validate --schema schemas/event.json --instance fixtures/events/NN-short-name.json`.
4. Add the file path to `tests/test_roundtrip.py::test_all_fixtures` in the SDK if it should also pass signature roundtrip.

Per [§4.2 Phase 0](../../concept-and-spec.md#42-build-order--what-claude-code-should-build-first), the target is 50 fixtures covering all `action.type` values and all canonicalization edge cases (deeply nested objects, supplementary-plane Unicode, very long strings, numeric edge cases). Phase 0 currently has 4 — the remaining 46 land alongside JCS edge-case test cases as the CTS comes online.
