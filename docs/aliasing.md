# Aliasing: map your own variables into a chain

Keystone does not dictate your field names. A link's `preimage` is your JSON, so
you name its fields after your own domain. The `@name` marker is the **alias** that
wires any field to an upstream link, so you map your variables into the chain
without changing how it composes. This is the piece that lets keystone model *your*
data, not a fixed schema.

## Your field names, your choice

There are no reserved field names in a preimage. Call things what your system
calls them. The only special value is the `@name` alias.

```json
{
  "chain": [
    { "name": "buyer",   "preimage": { "who": "did:key:zBuyer", "allowance": "payments" } },
    { "name": "grant",   "preimage": { "granted_by": "@buyer", "ceiling_usd": 500 } },
    { "name": "charge",  "preimage": { "under_authority": "@grant", "paid_for": "invoice/42", "result": "COMMITTED" } },
    { "name": "verdict", "preimage": { "covers": ["@buyer", "@grant", "@charge"], "trust": "TRUSTED" } }
  ]
}
```

Every field here is domain-chosen (`who`, `allowance`, `granted_by`, `ceiling_usd`,
`under_authority`, `paid_for`, `result`, `covers`, `trust`). None of them are
keystone keywords. What ties the chain together is the aliasing: `@buyer`,
`@grant`, `@charge`.

## How the alias resolves

Anywhere inside a preimage, `"@name"` is an alias for an earlier link. Before the
preimage is canonicalized and hashed, the verifier replaces each `"@name"` with
that link's already-computed reference (see [Chains](chains.md#composition-the-name-marker)).
So `grant`'s `granted_by` field carries the exact reference of `buyer`, and the
chain binds even though you chose the field name.

Verifying the chain above:

```
keystone chain: buyer  ->  grant  ->  charge  ->  verdict
------------------------------------------------------------
  OK  buyer
      sha256:f6dcd6e2...
  OK  grant
      sha256:3c404afb...  binds buyer
  OK  charge
      sha256:211358bb...  binds grant
  OK  verdict
      sha256:a309502b...  binds buyer, grant, charge
------------------------------------------------------------
KEYSTONE VALID: 4/4 links compose, recompute byte-for-byte, no issuer contact.
  capped: verdict is one verdict over [buyer, grant, charge] in order.
```

The `binds` column is the map you built: `grant` binds `buyer`, `charge` binds
`grant`, `verdict` binds all three. Your variable names, keystone's composition.

## Aliases work in nested objects and lists

`@name` is resolved everywhere in the preimage, not just at the top level. In the
example, `verdict`'s `covers` is a list of aliases:

```json
"covers": ["@buyer", "@grant", "@charge"]
```

Each entry resolves to the corresponding upstream reference, so one field can map
to many links at once (this is how a cap aggregates the chain).

## Wire incrementally with a stand-in alias

While building a downstream link you may not have the upstream one yet. `synth_ref`
gives you a deterministic, content-addressed stand-in you can alias against, then
swap for the real link later:

```python
from algovoi_keystone_connect import synth_ref
grant_stub = synth_ref("grant", ceiling_usd=500)   # a stable placeholder reference
```

## Rules

- An alias must point at an **earlier** link. A `@name` that names a later link, or
  one that does not exist, is a structural error (the chain is rejected, exit `2`).
- Link `name`s must be unique within the chain.
- Field names carry no meaning to the verifier; only the `@name` values and the
  resulting bytes matter. Two chains with different field names but the same
  resolved bytes produce the same references.

Because field naming is free and only the aliases and bytes matter, you can map an
existing domain model onto a keystone chain without reshaping your data: name the
fields as you already do, and alias the relationships with `@name`.
