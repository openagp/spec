# AGP JSON Schemas — v0.1

Draft 2020-12 JSON Schemas for every AGP message kind. These are the **canonical** definitions; reference SDKs bundle copies for runtime validation.

## Files

| File | Message kind | Flow |
|---|---|---|
| [`common.json`](common.json) | Shared definitions (`$defs/...`) reused by the others | — |
| [`event.json`](event.json) | Canonical signed event | A (L1) |
| [`policy.json`](policy.json) | Customer-authored policy descriptor | B (L2) |
| [`decision-request.json`](decision-request.json) | Vendor → plane synchronous decision request | C (L3) |
| [`decision-response.json`](decision-response.json) | Plane → vendor synchronous decision response | C (L3) |
| [`discovery.json`](discovery.json) | `.well-known/agp` capability + key discovery | — |
| [`actor.json`](actor.json) | Signed registry entry for a vendor / plane / customer | — |

## How references work

`event.json`, `policy.json`, etc. reference shared types via `common.json#/$defs/...`. Validators MUST resolve these locally by loading both files; do not assume `https://openagp.io/schemas/...` is a fetchable URL today (the host is reserved but not yet serving).

## Receiver behavior on unknown fields

Per spec [§3.9](../concept-and-spec.md#39-backward-compatibility-versioning-and-deprecation), receivers MUST ignore unknown fields rather than rejecting messages. The schemas therefore deliberately omit `additionalProperties: false`. SDKs SHOULD log a warning when they encounter unknown fields, but MUST NOT fail the message on that basis alone.

## Signature canonicalization

The `signature` field's wire form is defined in [`common.json#/$defs/signature`](common.json). The cryptographic protocol — how to produce the bytes that get signed and how to verify them — is specified in [ADR 0001](../decisions/0001-signature-canonicalization.md). The schema does **not** enforce the canonical signing protocol; that is the SDK's responsibility.

## Stability guarantees

- Within v0.1: schemas are frozen at this commit. Any change is either editorial (ADR not required) or substantive (ADR required, 2-week comment period per [GOVERNANCE.md](https://github.com/openagp/.github/blob/main/GOVERNANCE.md)).
- v0.1 → v0.2: additive only. New optional fields, new enum values. Existing fields cannot change semantics. Receivers built against v0.1 will continue to work on v0.2 messages.
- Major version bumps (v0.x → v1.0): may break. 12-month deprecation runway with both versions supported.

## Validation

To validate a fixture against a schema locally:

```bash
pip install jsonschema
python -m openagp.tools.validate \
  --schema schemas/event.json \
  --instance fixtures/events/01-tool-call-allowed.json
```

CI runs this against every fixture on every PR (see [`.github/workflows/validate.yml`](../.github/workflows/validate.yml)).
