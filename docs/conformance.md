# Conformance

The value of a content-addressed proof is that anyone can recompute it and get the
same answer. This SDK ships two implementations, Python and Node, and a self-test
that proves they agree byte for byte and that both match the published reference
vectors.

## Two implementations, one answer

| Runtime | Package | Command |
| --- | --- | --- |
| Python | `algovoi-keystone-compose` | `keystone-compose verify chain.json` |
| Node | `@algovoi/keystone` | `keystone verify chain.json` |

Both compute every link's reference as `"sha256:" + SHA-256(RFC 8785 JCS(preimage))`.
RFC 8785 (JCS) is the reason they can agree: it fixes a single canonical byte
string for any JSON value, independent of language, key insertion order, or
whitespace. If two implementations both honor JCS, they produce identical
references for identical inputs.

## Run the self-test

Python:

```bash
python conformance/run.py
```

Node twin, over the same example vectors:

```bash
node conformance/run.mjs
```

The Python runner checks that:

1. The golden chain is valid and capped at `trust_query_ref`.
2. Every reference it computes matches the published `keystone_v1` composition
   from [algovoi-jcs-conformance-vectors](https://github.com/chopmob-cloud/algovoi-jcs-conformance-vectors).
3. The tampered vectors are correctly rejected.

Running the Node twin over the same examples proves Python equals Node.

## The example vectors

Every capability has a golden vector and at least one broken counterpart, so the
suite exercises both the accept and the reject path.

| Vector | Exercises | Expected |
| --- | --- | --- |
| [`keystone-golden.json`](../examples/keystone-golden.json) | The canonical chain and the cap | valid |
| [`broken-execution.json`](../examples/broken-execution.json) | A tampered link | broken (link mismatch) |
| [`delegation-golden.json`](../examples/delegation-golden.json) | `field_equals`, `scope_subset`, `time_within` | valid |
| [`delegation-broken-scope.json`](../examples/delegation-broken-scope.json) | Scope widened across a boundary | broken (`scope_subset`) |
| [`delegation-broken-expired.json`](../examples/delegation-broken-expired.json) | Action outside the window | broken (`time_within`) |
| [`delegation-broken-delegate.json`](../examples/delegation-broken-delegate.json) | Delegate not bound to passport | broken (`field_equals`) |
| [`revocation-golden.json`](../examples/revocation-golden.json) | `not_revoked` before revocation | valid |
| [`revocation-broken-after.json`](../examples/revocation-broken-after.json) | Acted at or after revocation | broken (`not_revoked`) |
| [`revocation-broken-revoker.json`](../examples/revocation-broken-revoker.json) | Revoker not the grantor | broken (`field_equals`) |
| [`revocation-multihop.json`](../examples/revocation-multihop.json) | Multi-hop cascade caught: C acts after A revoked the upstream grant | broken (`not_revoked`) |
| [`journey-golden.json`](../examples/journey-golden.json) | `journey` covers every hop | valid |
| [`journey-broken.json`](../examples/journey-broken.json) | A hop omitted from the journey | broken (`journey`) |
| [`passport_vectors.json`](../examples/passport_vectors.json) | Passport decode and verify | mixed |

## Interoperability

Because the golden chain's references match the published `keystone_v1` vectors,
a chain built and verified with this SDK interoperates with any other
implementation that targets the same vectors. The vectors, not any single
codebase, are the standard; this SDK is one conformant implementation of them.
