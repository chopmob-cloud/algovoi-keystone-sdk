# Quickstart

Zero to a verified keystone chain and a verified agent passport in five minutes.

## 1. Install

```bash
pip install algovoi-keystone-compose
```

That is enough to verify chains. To verify agent passports as well (which needs a
FIPS 206 / Falcon-1024 verifier), install the extra:

```bash
pip install "algovoi-keystone-compose[passport]"
```

The only base dependency is `rfc8785`. The verifier makes no network call.

## 2. Verify a chain

Grab the canonical golden chain shipped in the repo (or point at your own):

```bash
keystone-compose verify examples/keystone-golden.json
```

Expected output:

```
keystone chain: passport_ref  ->  mandate_ref  ->  policy_ref  ->  policy_bound_ref  ->  decision_ref  ->  execution_ref  ->  trust_query_ref
------------------------------------------------------------
  OK  passport_ref
      sha256:b3594e33...
  ...
  OK  trust_query_ref
      sha256:18fb601a...  binds passport_ref, mandate_ref, policy_bound_ref, decision_ref, execution_ref
------------------------------------------------------------
KEYSTONE VALID: 7/7 links compose, recompute byte-for-byte, no issuer contact.
  capped: trust_query_ref is one verdict over [...] in order.
```

Exit code `0`. The chain composes, every link recomputes byte for byte, and the
final `trust_query_ref` is the cap that binds the whole sequence.

## 3. Watch it catch tampering

Any single altered byte breaks the link it belongs to and every link downstream
that binds it:

```bash
keystone-compose verify examples/broken-execution.json
```

```
  BAD execution_ref
      ^ recomputed sha256:8a71a2a3... != declared sha256:f6e2bfc1...
KEYSTONE BROKEN at execution_ref: reference mismatch.
  every link that binds it is therefore not provable from these bytes.
```

Exit code `1`. This is the whole point: the proof is in the recomputation, not in
trusting a stored digest.

## 4. Verify a chain with authorization assertions

Structural composition proves the links bind. Assertions prove the chain means
what it claims: no permission was widened, the revoker was the grantor, the action
happened inside the authority window. The delegation example carries all three:

```bash
keystone-compose verify examples/delegation-golden.json
```

```
assertions:
  OK  field_equals
  OK  field_equals
  OK  scope_subset
  OK  scope_subset
  OK  time_within
------------------------------------------------------------
KEYSTONE VALID: 8/8 links compose ... + field-equals, scope-subset, time-within hold.
```

Try a broken variant to see an assertion fail while the links still recompute:

```bash
keystone-compose verify examples/delegation-broken-scope.json
```

```
KEYSTONE BROKEN: scope_subset failed -- execution_B.scope widens decision_B.scope: max_amount 2000 exceeds 500.
  the chain recomputes, but it does not prove what it claims (e.g. scope widened, or out of window).
```

See [Assertions](assertions.md) for every assertion kind.

## 5. Verify an agent passport (offline)

Inspect a credential without trusting it:

```bash
keystone-compose passport inspect <credential>
```

Verify it against a known issuer key, fully offline:

```bash
keystone-compose passport verify <credential> --key <KID>=<PUBLIC_KEY_B64>
```

```
agent_did : did:key:z6Mk...
issuer    : did:key:passport-<kid>  kid=<kid>
scopes    : pay:invoice
spend     : 2000000 microusd / per_day
expires   : 2026-09-21T00:00:00Z
------------------------------------------------------------
PASSPORT ACTIVE: signature verifies offline, not expired, not revoked (status=active).
```

Exit code `0` when active; `1` when expired, revoked, invalid or unverifiable
(fail-closed). See [Passports](passport.md) for resolving keys from a well-known
URL, revocation lists, and scope or agent pinning.

## 6. Use it from Python

```python
import json
from algovoi_keystone import verify_chain

with open("examples/keystone-golden.json") as f:
    chain = json.load(f)

result = verify_chain(chain)
print(result.ok)                       # True
print([l.name for l in result.links])  # the ordered link names
print(result.cap)                      # 'trust_query_ref'
```

Full surface in the [Python API](python-api.md).

## Next steps

- [CLI reference](cli.md) for every flag.
- [Chains](chains.md) to author your own chain.
- [Assertions](assertions.md) for the authorization proofs.
- [Conformance](conformance.md) to prove Python and Node agree byte for byte.
