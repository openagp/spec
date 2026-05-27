# openagp/spec

**The Agent Governance Protocol — specification, JSON Schemas, and reference fixtures.**

## What's here

- [`concept-and-spec.md`](concept-and-spec.md) — full v0.1 working draft (concepts + protocol spec + reference implementation plan + adoption strategy)
- `schemas/` — JSON Schemas for canonical event, policy descriptor, decision request/response, discovery document *(coming)*
- `fixtures/events/` — 50 example events covering all `action.type` values *(coming)*
- `fixtures/policies/` — 10 example policies covering all rule patterns *(coming)*
- `decisions/` — RFCs and decision records for substantive spec changes *(coming)*

## Status

**v0.1 — working draft.** The spec is functional enough to start a reference implementation against, but is not yet a final RFC. Items marked **[OPEN]** in the spec are decisions for the standards working group once formed.

See [§4 of the spec](concept-and-spec.md#4-reference-implementation-plan) for the build order. The reference implementations live in sibling repos:

- [`openagp/sdk-python`](https://github.com/openagp/sdk-python)
- [`openagp/sdk-typescript`](https://github.com/openagp/sdk-typescript)
- [`openagp/cts`](https://github.com/openagp/cts) — Conformance Test Suite
- [`openagp/registry`](https://github.com/openagp/registry) — Public actor registry
- [`openagp/examples`](https://github.com/openagp/examples)

## Contributing

See [CONTRIBUTING.md](https://github.com/openagp/.github/blob/main/CONTRIBUTING.md) and [GOVERNANCE.md](https://github.com/openagp/.github/blob/main/GOVERNANCE.md) at the org level. For routing — questions vs. bugs vs. spec changes — see [SUPPORT.md](https://github.com/openagp/.github/blob/main/SUPPORT.md).

For spec changes:
1. Open a [Spec change issue](https://github.com/openagp/spec/issues/new?template=spec_change.yml) describing the change and rationale.
2. PR adds a decision record in `decisions/`.
3. Substantive changes have a 2-week comment period; breaking changes have 4 weeks.

### Local checks before opening a PR

```bash
# Validate every event fixture against schemas/event.json (same script CI runs)
pip install "jsonschema>=4.21.0" "referencing>=0.34.0"
# see .github/workflows/validate.yml for the exact validator

# Run the cross-language test vectors against an SDK
cd ../sdk-python && pytest tests/test_interop.py
cd ../sdk-typescript && npm test -- interop
cd ../cts && make vectors
```

If you're changing a schema or a test vector, **the SDKs and `agp-cts` must all stay green** — they share these inputs.

### Questions

Use [GitHub Discussions on this repo](https://github.com/openagp/spec/discussions) for design conversations and "how do I model X?" questions.

## License

[CC BY 4.0](LICENSE) — you may share and adapt the spec for any purpose, including commercially, with attribution.
