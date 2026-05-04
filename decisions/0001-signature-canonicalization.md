# ADR 0001 — Signature canonicalization

- **Status:** Accepted
- **Date:** 2026-05-04
- **Affects:** [`concept-and-spec.md` §3.4](../concept-and-spec.md#34-canonical-event-schema-flow-a--l1), §3.10, §6.2 Q1
- **Supersedes:** —
- **Deciders:** OpenAGP authors (Zeron founding team)

---

## Context

Every AGP message that carries authority — signed events (Flow A), signed policies (Flow B), signed decision requests/responses (Flow C) — must produce **identical signature bytes** on every conformant implementation, in every language, regardless of which JSON serializer constructed the wire form. Without a precisely-specified canonicalization scheme, signatures will not interoperate, and AGP's entire trust model fails silently.

Spec [§6.2 Q1](../concept-and-spec.md#62-open-questions) flagged this as the single most consequential cryptographic decision in v0.1. This ADR resolves it.

## Decision

AGP v0.1 uses **RFC 8785 JSON Canonicalization Scheme (JCS) + Ed25519 detached signatures**, with the signing input bound to `key_id` and `alg` to prevent algorithm-substitution attacks.

### The canonical signing protocol (precise)

For any message type carrying a `signature` object (events, policies, decision requests, decision responses):

#### To sign

1. Construct the message as a JSON object. Set:

   ```json
   "signature": {
     "key_id": "<key_id>",
     "alg": "Ed25519"
   }
   ```

   Note: **the `value` field is absent** at signing time. `key_id` and `alg` are present. This binds the signature metadata to the bytes being signed.

2. Apply **RFC 8785 JCS** to the entire message object to produce the *canonical signing input* — a UTF-8 byte sequence with deterministic property ordering, no insignificant whitespace, and a precisely-specified number representation.

3. Compute the signature: `sig_bytes = Ed25519.sign(private_key, canonical_signing_input)`.

4. Base64-encode `sig_bytes` (standard Base64, RFC 4648 §4 — **not** Base64URL) and place it at `signature.value`.

5. Serialize the message for transport. Wire serialization MAY differ from the canonical form (e.g., an HTTP server may emit different whitespace) — this does not affect signature validity, because verification re-canonicalizes.

#### To verify

1. Parse the received message as a JSON object. Reject if not a well-formed JSON object.
2. Validate the message against its JSON Schema. Reject on schema failure.
3. Extract `signature.value`, then **remove the `value` field** from the in-memory `signature` object so only `{key_id, alg}` remain.
4. Apply RFC 8785 JCS to the message to produce the canonical signing input.
5. Look up the public key by `signature.key_id` (via the AGP Registry, or the actor's `.well-known/agp` discovery document — registry is authoritative).
6. Reject if `signature.alg != "Ed25519"`. AGP v0.1 supports only Ed25519; any other value is a rejection, not a fallback.
7. Compute `Ed25519.verify(public_key, base64_decode(signature.value), canonical_signing_input)`. Reject on failure.
8. The message is authenticated. The verifier MAY then apply transport-level checks (`occurred_at` skew bounds, replay-cache lookup on `event_id`, etc.) — those are receiver policies, outside this ADR.

#### Pseudocode (Python-ish)

```python
def sign(message: dict, private_key: bytes, key_id: str) -> dict:
    msg = deepcopy(message)
    msg.pop("signature", None)
    msg["signature"] = {"key_id": key_id, "alg": "Ed25519"}
    canonical = jcs.canonicalize(msg)              # RFC 8785, UTF-8 bytes
    sig = ed25519.sign(private_key, canonical)
    msg["signature"]["value"] = base64.b64encode(sig).decode("ascii")
    return msg

def verify(message: dict, public_key: bytes) -> None:
    sig_obj = message.get("signature")
    if not sig_obj or sig_obj.get("alg") != "Ed25519":
        raise InvalidSignature("missing or unsupported signature.alg")
    sig_value = sig_obj["value"]
    msg_for_verify = deepcopy(message)
    msg_for_verify["signature"] = {
        "key_id": sig_obj["key_id"],
        "alg": sig_obj["alg"],
    }
    canonical = jcs.canonicalize(msg_for_verify)
    ed25519.verify(public_key, base64.b64decode(sig_value), canonical)
```

### Properties bound by the signature

The signature covers, byte-for-byte after canonicalization:

- The full message body (every field every receiver might act on).
- `signature.key_id` — prevents substituting a different signer's key after the fact.
- `signature.alg` — prevents algorithm-substitution attacks if v0.2+ adds additional `alg` values.

The signature does **not** cover:

- `signature.value` itself (would be self-referential).
- Transport headers (HTTP `Authorization`, `Date`, etc.). Those are protected by mTLS / TLS at the connection layer, not by the message signature.

## Alternatives considered

### A. JSON Web Signatures (JWS, RFC 7515) — *rejected*

JWS is the industry default for signed JSON. We rejected it for v0.1 on three grounds:

1. **Wire shape mismatch.** JWS emits `<base64url(header)>.<base64url(payload)>.<base64url(signature)>`, which is opaque to any reader without a JWS parser. AGP events are read by humans, ledgers, and audit tools — keeping the JSON inspectable matters.
2. **Implementation surface.** JWS has many cryptosuites and a permissive header model that has historically produced critical vulnerabilities (alg=none, key confusion, header substitution). JCS + Ed25519 has a tiny attack surface.
3. **Cross-language drift risk.** JWS libraries differ subtly across languages on canonicalization of the protected header. JCS is a precise canonicalization, end of story — same bytes everywhere.

JWS-detached payload was also considered. Its mode of "header is base64url-protected, payload is unencoded raw bytes" reduces opacity but inherits JWS's other risks. Not adopted.

### B. Stripped JCS — sign over the message with `signature` removed entirely — *rejected*

This is simpler than the chosen approach. Rejected because it does not bind `key_id` or `alg` into the signed bytes, opening the door to algorithm-substitution attacks if v0.2+ adds a second algorithm. The chosen approach pays one extra step (re-insert `{key_id, alg}` before canonicalizing) for permanent forward-safety.

### C. Detached signatures in a sibling field — *rejected*

E.g., wrap the signed object in `{"payload": <event>, "signature": {...}}` instead of embedding `signature` inside. Cleaner separation, but every reader (ledgers, audit tools, dashboards) would have to unwrap the envelope. Embedding `signature` inside the message keeps the on-the-wire shape and the in-memory shape identical — preserved fidelity is more valuable than envelope cleanliness.

## Consequences

### Positive

- **Inspectable.** A signed AGP event is still a JSON object that auditors and humans can read directly. No base64url decoding required to see the action that was performed.
- **Cross-language interoperability is mechanical.** RFC 8785 has reference implementations in Python (`rfc8785`), TypeScript (`canonicalize`), Go (`gowebpki/jcs`), Rust, Java. Each language's CTS run produces identical canonical bytes.
- **Tiny crypto surface.** One algorithm (Ed25519). One canonicalization (JCS). No options. Nothing to negotiate.
- **Forward-compatible.** Adding `alg` values in v0.2 is straightforward — the `alg` field is already in the signed input, and receivers reject unknown `alg` values cleanly.

### Negative

- **No off-the-shelf JWS toolkits.** Implementers cannot reach for `jose` or `pyjwt`. They must compose JCS + Ed25519 + Base64 themselves (or use the OpenAGP reference SDKs, which is the intent).
- **JCS edge cases need test fixtures.** RFC 8785's number canonicalization (e.g., trailing zeros, exponent normalization) is precise but easy to get subtly wrong. The CTS MUST include fixtures that exercise these edges (numbers with high precision, Unicode normalization, deeply nested objects, large arrays). [§4.2 Phase 0](../concept-and-spec.md#42-build-order--what-claude-code-should-build-first) item 4 — the 50 fixture events — must include canonicalization edge cases.
- **No streaming / partial verification.** The whole canonical form must be in memory to verify. Non-issue at AGP message sizes (events are kilobytes, not megabytes); flagged so nobody assumes streaming verification later.

### Neutral / for future ADRs

- **Key rotation:** out of scope for this ADR. Handled via `key_id` resolution at the registry — this ADR only specifies how signatures are computed and verified for a given resolved key. Registry-side key rotation semantics: future ADR.
- **Multiple signatures:** v0.1 supports one signature per message. Multi-signature (e.g., a vendor-co-signed-by-an-auditor event) is a future ADR; the schema reserves `signature` as a single object now and would migrate to `signatures: [...]` in a future major version, additively.
- **Signing of transport headers:** out of scope; transport security is TLS / mTLS at the connection.

## Notes on RFC 8785 implementation

JCS implementations MUST:

- Sort object keys lexicographically by UTF-16 code unit (RFC 8785 §3.2.3). Note: this is **UTF-16 code units**, not Unicode code points. Most JSON-canonicalization libraries handle this correctly.
- Normalize numbers per RFC 8785 §3.2.2 / ECMAScript `Number.prototype.toString` semantics. Integers print without decimals; non-integers print in shortest-round-trip form. Implementations MUST NOT permit a JSON number representation to vary across runs (no `1e2` vs `100.0` differences after canonicalization).
- Emit UTF-8 with no BOM, no insignificant whitespace, and the JSON string-escaping rules from RFC 8785 §3.2.4.
- Reject non-finite numbers (`NaN`, `±Infinity`) — these are not valid JSON and must not appear in any AGP message.

The Conformance Test Suite ([`openagp/cts`](https://github.com/openagp/cts), Phase 2) MUST include fixtures that exercise these edges. An implementation that passes JCS canonicalization on simple objects but fails on (e.g.) Unicode-supplementary-plane keys is not conformant.

## References

- [RFC 8785 — JSON Canonicalization Scheme](https://datatracker.ietf.org/doc/html/rfc8785)
- [RFC 8032 — Edwards-Curve Digital Signature Algorithm (EdDSA)](https://datatracker.ietf.org/doc/html/rfc8032) — Ed25519 specification
- [RFC 4648 §4 — Base64](https://datatracker.ietf.org/doc/html/rfc4648#section-4)
- Reference implementations:
  - Python: [`rfc8785` on PyPI](https://pypi.org/project/rfc8785/)
  - TypeScript / JavaScript: [`canonicalize` on npm](https://www.npmjs.com/package/canonicalize)
  - Go: [`gowebpki/jcs`](https://github.com/gowebpki/jcs)
