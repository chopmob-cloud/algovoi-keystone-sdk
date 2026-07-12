# algovoi-keystone

**One command to prove an agentic-action keystone chain composes end to end.**

Everyone has receipts. The keystone is the *composition*: the proof that
identity binds to authority binds to policy binds to the decision binds to the
execution binds to one trust verdict, every link a content address, recomputable
offline, with no issuer contact. A plain receipt checker recomputes individual
digests. This recomputes the whole chain and proves it *binds*.

```
keystone-compose verify chain.json
```

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

Tamper any link and that link, and every link that binds it, diverge:

```
  BAD execution_ref
      ^ recomputed sha256:8a71a2a3... != declared sha256:f6e2bfc1...
KEYSTONE BROKEN at execution_ref: ... every link that binds it is therefore not provable from these bytes.
```

(`keystone-compose verify` exits 0 if the chain composes, 1 if any link is broken, 2 on a malformed chain.)

## Why this exists

As agent governance matures from "here is a signed receipt" to "prove the whole
chain composed, that the decision that authorized is the one that executed is the
one the verdict covers", a receipt is no longer enough. The keystone is that
proof, and it is the one piece of the agentic-governance stack that cannot be
reproduced by cloning a single digest construction, because it requires the whole
composition. This is the turnkey verifier for it.

## The chain format

A keystone chain is an ordered list of links. Each link's reference is
`"sha256:" + SHA-256(RFC 8785 (JCS)(preimage))`. Composition is **structural**: a
preimage refers to an earlier link by `@name`, so a downstream reference cannot be
computed without the exact upstream reference.

```json
{
  "schema": "algovoi-keystone-chain/v1",
  "canon": "jcs-rfc8785-v1",
  "chain": [
    { "name": "passport_ref",  "preimage": { "agent_id": "agent-001", "...": "..." } },
    { "name": "decision_ref",  "preimage": { "agent_ref": "@passport_ref", "mandate_ref": "@mandate_ref", "policy_bound_ref": "@policy_bound_ref", "verdict": "ALLOW" } },
    { "name": "execution_ref", "preimage": { "decision_ref": "@decision_ref", "outcome": "COMMITTED", "...": "..." } },
    { "name": "trust_query_ref", "preimage": { "subject_refs": ["@passport_ref", "@mandate_ref", "@policy_bound_ref", "@decision_ref", "@execution_ref"], "trust_outcome": "TRUSTED" } }
  ]
}
```

An optional `"ref"` on a link is the declared output; the verifier recomputes and
checks it. See [`examples/keystone-golden.json`](examples/keystone-golden.json)
for the canonical chain (its values match the published `keystone_v1` composition
in [algovoi-jcs-conformance-vectors](https://github.com/chopmob-cloud/algovoi-jcs-conformance-vectors)).

## Install

```bash
pip install algovoi-keystone-compose      # Python:  keystone-compose verify chain.json
npm  install -g @algovoi/keystone         # Node:    keystone verify chain.json
```

Verifying agent passports (not chains) additionally needs a FIPS 206 verifier:

```bash
pip install "algovoi-keystone-compose[passport]"
```

Python and Node produce byte-identical references on every link. RFC 8785 JCS +
SHA-256 is the whole dependency; the verifier makes no network call.

## Documentation

Full docs live in [`docs/`](docs/):

| Doc | Covers |
| --- | --- |
| [Overview](docs/index.md) | The keystone model, and a map of every option |
| [Quickstart](docs/quickstart.md) | Zero to a verified chain and a verified passport in five minutes |
| [CLI reference](docs/cli.md) | Every `keystone-compose` command and flag, with exit codes |
| [Chains](docs/chains.md) | The chain file format, `@name` composition, refs, the cap |
| [Assertions](docs/assertions.md) | All five assertion kinds and the scope-containment model |
| [Passports](docs/passport.md) | Present, inspect and verify agent passports offline |
| [Python API](docs/python-api.md) | Every public function and dataclass |
| [Conformance](docs/conformance.md) | Python and Node parity, and the example vectors |

## License

Apache-2.0. (c) AlgoVoi. Keep the NOTICE attribution when you redistribute.
