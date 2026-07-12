# Execution and decision refs

`algovoi-execution-ref` builds the references for the decision and execution
stages of a keystone chain, and can bind a whole keystone into one self-describing
record. It is an **independent** package: install and use it on its own.

```bash
pip install algovoi-execution-ref
```

## The ref builders

Every builder returns `"sha256:" + SHA-256(RFC 8785 JCS(preimage))`, so its output
is a keystone reference that composes with the rest of the chain.

### `decision_ref(agent_ref, mandate_ref, policy_bound_ref, verdict) -> str`

The pre-action decision reference: the keystone that an execution binds to. Binds
the acting agent, the mandate (authority), the bound policy, and the verdict.

### `execution_ref(decision_ref, action_type, scope, outcome, executed_at_ms) -> str`

The execution reference, bound to the decision that authorized it.

```python
from algovoi_execution_ref import decision_ref, execution_ref

d = decision_ref("sha256:...agent", "sha256:...mandate", "sha256:...policy", "ALLOW")
e = execution_ref(d, "payment", "invoice/42", "COMMITTED", 1716460800000)
```

- Every `*_ref` input must be a valid keystone reference: `sha256:` followed by 64
  lowercase hex characters (the output of `keystone_ref`). A malformed ref raises
  `ExecutionRefError`.
- `scope` is a non-empty string.
- `outcome` is one of `COMMITTED`, `SKIPPED`, `FAILED`, `REVERSED`.
- `executed_at_ms` is an integer millisecond timestamp.

### `trust_query_ref(subject_refs, trust_outcome) -> str`

A composite trust verdict over an ordered list of references (the cap of a chain).

### `execution_binding(execution_ref, settlement_ref, retention_chain_ref) -> str`

A post-settlement accountability binding over an execution, tying it to
settlement and a retention chain.

## Bind a whole keystone at once

### `bind_keystone(*, passport_ref, mandate_ref, policy_bound_ref, verdict, action_type, scope, outcome, executed_at_ms, trust_outcome) -> dict`

Compute the decision, execution and trust-query refs together and return one
self-describing record:

```python
from algovoi_execution_ref import bind_keystone

record = bind_keystone(
    passport_ref="sha256:...",
    mandate_ref="sha256:...",
    policy_bound_ref="sha256:...",
    verdict="ALLOW",
    action_type="payment",
    scope="invoice/42",
    outcome="COMMITTED",
    executed_at_ms=1716460800000,
    trust_outcome="TRUSTED",
)
```

## Verify a record

### `verify_record(record) -> bool`

Recompute the decision, execution and trust-query refs (and the chain among them)
from a record's own fields and return whether they all hold. Fully offline.

```python
from algovoi_execution_ref import verify_record
assert verify_record(record)
```

`ExecutionRefError` is raised when inputs violate the discipline (for example a
non-string scope, or an outcome outside the allowed set).

## Retention chain

`algovoi-retention-chain` builds an append-only, tamper-evident chain of receipt
hashes, the structure `execution_binding` points at through its
`retention_chain_ref`. Each link binds the previous receipt hash, so a broken or
reordered link is detected.

```python
from algovoi_retention_chain import retention_chain_ref, verify_chain_link, verify_chain_sequence

ref = retention_chain_ref(
    prev_receipt_hash="sha256:...", receipt_hash="sha256:...",
    chain_seq=1, issuer_id="my-issuer",
)
verify_chain_link(chain_ref=ref, prev_receipt_hash="sha256:...",
                  receipt_hash="sha256:...", chain_seq=1, issuer_id="my-issuer")  # True = intact
verify_chain_sequence(records)   # True when the whole sequence is an unbroken chain
```

## CLI

```
algovoi-execution-ref [--verify] [file]
```

Reads a record (from `file` or stdin), binds an execution to the keystone, and
prints the result. With `--verify`, it verifies a record from stdin instead and
exits `0` if consistent, `1` if not.
