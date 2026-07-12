# Chains

A keystone chain is an ordered list of content-addressed links that bind into one
recomputable sequence. This page is the reference for the file format, how
references are computed, and how composition works.

## File format

```json
{
  "schema": "algovoi-keystone-chain/v1",
  "canon": "jcs-rfc8785-v1",
  "chain": [
    { "name": "<link name>", "preimage": { ... }, "ref": "sha256:..." }
  ],
  "assertions": [ { "kind": "...", ... } ]
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `chain` | yes | A non-empty array of links, in order. |
| `assertions` | no | Semantic checks over the links. See [Assertions](assertions.md). |
| `schema`, `canon`, `description` | no | Informational. The verifier does not depend on them. |

Each link is:

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | A unique name within the chain. Later links reference it as `@name`. |
| `preimage` | yes | The JSON object being committed to. Its canonical bytes are what get hashed. |
| `ref` | no | The declared reference. If present, the verifier recomputes and checks it. If absent, the verifier computes it and carries it forward. |

Duplicate names, a missing `name`, a missing `preimage`, or a `@name` pointing at
a link that is not earlier in the chain are all structural errors (exit `2`).

## How a reference is computed

For any preimage object:

```
ref = "sha256:" + hex( SHA-256( RFC 8785 JCS( preimage ) ) )
```

RFC 8785 (JCS) canonicalization sorts object keys and emits a single deterministic
byte string, so the same preimage produces the same reference in any language or
runtime. This is the property the [conformance](conformance.md) suite proves
between Python and Node.

## Composition: the `@name` marker

Composition is structural. Anywhere inside a preimage, the string `"@name"` is a
reference to an earlier link. Before the preimage is canonicalized and hashed, the
verifier replaces every `"@name"` with that earlier link's already computed
reference.

```json
{ "name": "decision_ref",
  "preimage": {
    "agent_ref": "@passport_ref",
    "mandate_ref": "@mandate_ref",
    "policy_bound_ref": "@policy_bound_ref",
    "verdict": "ALLOW"
  }
}
```

Because `decision_ref`'s bytes include the exact reference of `passport_ref`,
`mandate_ref` and `policy_bound_ref`, you cannot compute `decision_ref` without
first computing those three exactly. Change any upstream preimage by one byte and
its reference changes, so `decision_ref` changes, and so does everything that
binds `decision_ref`. The tamper cascades downstream and is detected at the first
divergence.

`@name` works inside nested objects and lists. A link's `binds` set (shown in the
CLI output) is every distinct `@name` its preimage referenced.

## The cap

If the final link binds two or more prior links, it is the **cap**: a single
reference that aggregates the whole chain into one verdict. In the canonical
chain, `trust_query_ref` is the cap:

```json
{ "name": "trust_query_ref",
  "preimage": {
    "subject_refs": ["@passport_ref", "@mandate_ref", "@policy_bound_ref", "@decision_ref", "@execution_ref"],
    "trust_outcome": "TRUSTED"
  }
}
```

One reference now stands for the entire composed chain, in order. A relying party
can carry that single cap reference and, given the underlying links, recompute the
whole proof.

## Worked example: the canonical keystone

[`examples/keystone-golden.json`](../examples/keystone-golden.json) is the
reference chain:

```
passport_ref  ->  mandate_ref  ->  policy_ref  ->  policy_bound_ref  ->  decision_ref  ->  execution_ref  ->  trust_query_ref
```

It models a governed agent action end to end:

1. `passport_ref` identity of the acting agent.
2. `mandate_ref` the standing authority (a spend mandate).
3. `policy_ref` the policy in force.
4. `policy_bound_ref` that policy bound to a subject.
5. `decision_ref` the ALLOW decision, binding agent, mandate and bound policy.
6. `execution_ref` the committed execution, binding the decision.
7. `trust_query_ref` the cap: a TRUSTED verdict over all of the above.

Its link references match the published `keystone_v1` composition in the
[algovoi-jcs-conformance-vectors](https://github.com/chopmob-cloud/algovoi-jcs-conformance-vectors)
project, so a chain built with this SDK interoperates with anything else that
targets those vectors.

Verify it:

```bash
keystone-compose verify examples/keystone-golden.json
```

## Structural composition is necessary but not sufficient

A chain that recomputes proves the links are internally consistent and bind. It
does **not**, on its own, prove the chain is *authorized*: that no permission was
widened, that the party who revoked was the party who granted, that the action
fell inside its authority window. Those are semantic facts, expressed as
[assertions](assertions.md), and the verifier checks them alongside the structure.
