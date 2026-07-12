# Assertions

Structural composition proves a chain's links bind. **Assertions** prove the chain
means what it claims. They live in the top-level `assertions` array and are checked
after every link recomputes. A chain is valid only when every link recomputes
**and** every assertion holds.

```json
{
  "chain": [ ... ],
  "assertions": [
    { "kind": "field_equals", "a": "delegation_AB.delegator_id", "b": "passport_A.agent_id" },
    { "kind": "scope_subset", "inner": "decision_B", "outer": "delegation_AB" }
  ]
}
```

An assertion that references a link, field or window that is not in the chain is a
structural error (exit `2`). An assertion that is well-formed but does not hold is
a broken chain (exit `1`) with the message `KEYSTONE BROKEN: <kind> failed -- <detail>`.

There are five kinds.

## `scope_subset`

Prove no permission was widened across a boundary: the inner link's `scope` is
wholly contained by the outer link's `scope`.

```json
{ "kind": "scope_subset", "inner": "<link name>", "outer": "<link name>" }
```

Both links must carry a `scope` field. Containment is evaluated by the rules in
[The scope-containment model](#the-scope-containment-model) below. This is the
assertion that catches privilege escalation: a delegate that tries to act beyond
what it was delegated, or an execution that exceeds the decision that authorized
it.

Chaining two `scope_subset` assertions proves a whole delegation stayed bounded:

```json
{ "kind": "scope_subset", "inner": "decision_B",  "outer": "delegation_AB" },
{ "kind": "scope_subset", "inner": "execution_B", "outer": "decision_B" }
```

execution within decision within delegation: nothing widened at any hop.

## `field_equals`

Prove two fields across the chain are equal. This binds parties together: the
delegator named in a delegation is the agent whose passport is in the chain; the
revoker is the original grantor.

```json
{ "kind": "field_equals", "a": "<link>.<field>", "b": "<link>.<field>" }
```

Both operands are `link.field` paths. The assertion holds when the two field
values are equal. Example, binding a delegation to the two agents' passports:

```json
{ "kind": "field_equals", "a": "delegation_AB.delegator_id", "b": "passport_A.agent_id" },
{ "kind": "field_equals", "a": "delegation_AB.delegate_id",  "b": "passport_B.agent_id" }
```

## `time_within`

Prove a timestamp fell inside an authority window.

```json
{ "kind": "time_within", "time": "<link>.<field>", "window": "<link name>" }
```

`time` is a `link.field` path to an integer millisecond timestamp. `window` names
a link whose preimage carries integer `not_before_ms` and `not_after_ms` fields.
The assertion holds when `not_before_ms <= time <= not_after_ms`. All three values
must be integers, or the assertion fails closed.

Example: the execution happened inside the delegation's validity window.

```json
{ "kind": "time_within", "time": "execution_B.executed_at_ms", "window": "delegation_AB" }
```

## `not_revoked`

Prove an action happened strictly before the authority it relied on was revoked.

```json
{ "kind": "not_revoked", "action": "<link name>", "revocation": "<link name>", "time": "<link>.<field>" }
```

`revocation` names a link whose preimage carries an integer `revoked_at_ms`.
`time` is a `link.field` path to the action's integer millisecond timestamp. The
assertion holds when `time < revoked_at_ms`. An action at or after the revocation
instant acted under revoked authority and breaks the chain.

Because the revocation link structurally binds the revoked authority with an
`@name` reference (and a `field_equals` binds the revoker to the grantor), this is
a temporal cascade that holds even multi-hop: any action whose chain transitively
binds a revoked authority is caught. See
[`examples/revocation-multihop.json`](../examples/revocation-multihop.json).

## `journey`

Prove one reference stands for a whole multi-agent task: the journey link must
bind every declared hop, with none omitted.

```json
{ "kind": "journey", "ref": "<journey link name>", "hops": ["<link>", "<link>", ...] }
```

`ref` names the journey link. `hops` is the list of link names that the journey is
required to cover. The assertion holds when `hops` is non-empty and the journey
link's preimage binds (references with `@name`) every one of them, and each hop is
a real link in the chain. Any hop the journey does not cover fails the assertion.

Combined with `scope_subset`, `field_equals` and `not_revoked`, a single
`journey` reference proves an entire A to B to C task end to end: identity and
authority continuity, scope never widened, nothing acted under a revoked grant.
See [`examples/journey-golden.json`](../examples/journey-golden.json).

## The scope-containment model

`scope_subset` calls a containment check between an outer scope and an inner
scope. A scope is a JSON object with optional dimensions. An **absent dimension on
the outer scope does not constrain that dimension**. Where the outer scope does
constrain a dimension, the inner must demonstrably stay within it, fail-closed.

| Dimension | Rule | Fails when |
| --- | --- | --- |
| `resource` | Hierarchical path. The outer path must be a segment-wise prefix of the inner path (inner is the outer or a sub-path). | The inner resource is not under the outer resource. |
| `max_amount` | Numeric ceiling. Inner `max_amount` must be present and `<= outer max_amount`. | Inner is missing `max_amount`, or exceeds the outer. |
| `capabilities` | Set membership. Inner set must be a subset of the outer set. | Inner has a capability the outer does not grant. |
| `jurisdictions` | Set membership. Inner set must be a subset of the outer set. | Inner names a jurisdiction the outer does not allow. |

If either scope is not an object, containment reduces to equality.

Worked example. Outer (a delegation) grants:

```json
{ "resource": "payments/transfer", "max_amount": 1000, "jurisdictions": ["GB", "US"] }
```

Inner (the decision made under it) claims:

```json
{ "resource": "payments/transfer/usdc", "max_amount": 500, "jurisdictions": ["GB"] }
```

This is contained: `payments/transfer/usdc` is under `payments/transfer`, `500 <= 1000`,
and `{GB}` is a subset of `{GB, US}`. Widen any of those (a sibling resource, a
higher amount, an extra jurisdiction) and `scope_subset` fails.

## The four delegation vectors

The example set demonstrates the golden path and one failure per assertion class:

| Vector | Result |
| --- | --- |
| [`delegation-golden.json`](../examples/delegation-golden.json) | Valid: composes, and all `field_equals` / `scope_subset` / `time_within` hold. |
| [`delegation-broken-scope.json`](../examples/delegation-broken-scope.json) | `scope_subset` fails: a scope was widened across a boundary (the execution's `max_amount` exceeds the decision). |
| [`delegation-broken-expired.json`](../examples/delegation-broken-expired.json) | `time_within` fails: the execution fell outside the delegation window. |
| [`delegation-broken-delegate.json`](../examples/delegation-broken-delegate.json) | `field_equals` fails: the delegate did not match the passport. |

And the revocation set: [`revocation-golden.json`](../examples/revocation-golden.json),
[`revocation-broken-after.json`](../examples/revocation-broken-after.json) (acted
after revocation), [`revocation-broken-revoker.json`](../examples/revocation-broken-revoker.json)
(revoker was not the grantor), and [`revocation-multihop.json`](../examples/revocation-multihop.json)
(a multi-hop violation caught: C acts after A revoked the upstream grant).
