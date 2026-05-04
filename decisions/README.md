# Decision Records

This directory holds **AGP Decision Records (ADRs)** — the trail of substantive decisions made during spec authorship. Each ADR captures: the decision, the alternatives considered, the rationale, and the consequences.

## Numbering

ADRs are numbered sequentially (`0001-`, `0002-`, …) in the order they are accepted. Numbers are never reused; superseded decisions stay in place with a `Status: Superseded by 00NN` line, and the new decision links back.

## Status values

- `Proposed` — opened as a PR, comment period open
- `Accepted` — merged after the comment period
- `Rejected` — discussed and not adopted; kept in tree as a record
- `Superseded` — replaced by a later decision; kept for history

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-signature-canonicalization.md) | Signature canonicalization (JCS + Ed25519) | **Accepted** |

## Authoring

When you propose a substantive change to the spec, open it as an ADR PR:

1. Copy the template (or pattern of an existing ADR).
2. Number it `00NN` where `NN` is the next unused integer.
3. Set `Status: Proposed`.
4. Open the PR. Comment period: 2 weeks for substantive changes, 4 weeks for breaking ones (per [GOVERNANCE.md](https://github.com/openagp/.github/blob/main/GOVERNANCE.md)).
5. On merge, set `Status: Accepted` in the same commit that merges the corresponding spec change.
