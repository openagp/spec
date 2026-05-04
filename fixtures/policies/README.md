# Policy fixtures (Flow B)

Hand-authored canonical policy descriptors. Each one exercises a distinct matcher pattern in the v0.1 DSL. CI validates fixtures against [`schemas/policy.json`](../../schemas/policy.json); reference SDKs additionally evaluate them against test events to assert decisions.

## Inventory

| File | Matchers exercised | Default fallback | Demonstrates |
|---|---|---|---|
| [`01-block-external-email.json`](01-block-external-email.json) | equality (`tool_name`), `domain_not_in` (with wildcard) | `allow_with_log` | Single-rule scope, wildcarded allowlist |
| [`02-block-pii-outbound.json`](02-block-pii-outbound.json) | `domain_not_in`, `contains_pattern` (regex) | `allow_with_log` | Multiple AND'd matchers in one rule |
| [`03-log-database-writes.json`](03-log-database-writes.json) | `starts_with`, `annotate.scf_controls` | `allow_with_log` | `logged_only` decision; SCF tagging |
| [`04-vendor-allowlist.json`](04-vendor-allowlist.json) | `not_in` on `actor.vendor` | `block` | Vendor-scoped policy; default-block fallback |
| [`05-multi-rule-composite.json`](05-multi-rule-composite.json) | `domain_in`, `domain_not_in`, `starts_with` | `allow_with_log` | First-match-wins across 3 rules |

## v0.1 matcher reference

Within a rule's `when` block, each top-level key is a JSON-pointer-style path into the event (e.g. `action.tool_name`, `actor.vendor`). The value is either:

- A **literal**: the matcher is equality. `"action.tool_name": "email.send"` matches if `event.action.tool_name == "email.send"`.
- A **matcher object**: the value's keys are matcher names. v0.1 supports:

| Matcher | Argument | Semantics |
|---|---|---|
| `equals` / `eq` | any | Equality (alias for the literal form) |
| `not_equals` / `ne` | any | Inverse of equality |
| `in` | array | Field value is one of the array elements |
| `not_in` | array | Field value is none of the array elements |
| `starts_with` | string | Field value (string) starts with the argument |
| `ends_with` | string | Field value (string) ends with the argument |
| `contains_pattern` | regex string | RE2-style regex; matches `re.search` on the field |
| `domain_in` | array of host patterns | Field is a URL or email; its host matches one of the patterns. Patterns may be exact (`acme.com`) or wildcard (`*.acme.com`) |
| `domain_not_in` | array of host patterns | Inverse of `domain_in` |

A rule fires only when **all** of its `when` conditions match (AND semantics). Within a policy, **the first matching rule wins**; `then.decision` is the result. If no rule matches, the result is the policy's `fallback.decision` (or `allowed` if no fallback is declared).

## Adding new fixtures

1. Author `NN-short-name.json` numbered sequentially.
2. Add a row to the table above with the matchers it exercises.
3. Confirm it validates: `python -m openagp.tools.validate --kind policy --instance fixtures/policies/NN-short-name.json`.
4. Add a corresponding test case (policy + event + expected decision) to the SDK's policy test vectors.
