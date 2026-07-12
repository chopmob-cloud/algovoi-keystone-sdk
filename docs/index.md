# algovoi-keystone-compose

**Verify that an agentic-action keystone chain composes end to end, offline.**

Everyone has receipts. A receipt proves one fact happened and was signed. The
*keystone* is the composition: proof that identity binds to authority binds to
policy binds to the decision binds to the execution binds to one trust verdict,
every link a content address, the whole sequence recomputable offline with no
issuer contact.

A plain receipt checker recomputes individual digests. This recomputes the whole
chain and proves it *binds*: the decision that authorized is provably the one
that executed, is provably the one the final verdict covers. Tamper any link and
that link, plus every link that binds it, diverge.

```
keystone-compose verify chain.json
```

RFC 8785 JCS + SHA-256 is the entire trust base. No network call, no issuer
lookup, no signing key required to check a chain.

## The two things this SDK verifies

| Artifact | What it proves | Command |
| --- | --- | --- |
| **Keystone chain** | A sequence of content-addressed links composes and binds into one recomputable proof | `keystone-compose verify chain.json` |
| **Agent passport** | A Falcon-1024 credential (agent DID, scopes, spend bound, expiry) is authentic, unexpired and unrevoked, offline | `keystone-compose passport verify <credential>` |

The chain carries **composition and integrity**. The passport signature carries
**authenticity**. They are complementary layers, and they meet on one primitive:
a keystone reference is `"sha256:" + SHA-256(RFC 8785 JCS(preimage))`, the same
canonicalization on both sides.

## The keystone model in one page

A chain is an ordered list of **links**. Each link has a `name`, a `preimage`
(the JSON object being committed to), and an optional declared `ref`.

- **Reference.** Every link's reference is `"sha256:" + SHA-256(RFC 8785 JCS(preimage))`.
  RFC 8785 (JCS) canonicalization makes the bytes deterministic across
  implementations and languages, so the hash is reproducible anywhere.
- **Composition.** A preimage refers to an earlier link with the marker
  `"@name"`. Before hashing, `@name` is replaced by that earlier link's already
  computed reference. So a downstream reference **cannot** be computed without the
  exact upstream reference. This is what makes the chain structural rather than a
  bag of independent digests.
- **The cap.** If the final link binds two or more prior links, it is the *cap*:
  a single reference that aggregates the whole chain into one verdict, in order.
- **Assertions.** Beyond structural composition, a chain can carry semantic
  assertions (no permission widened, revoker equals grantor, action inside a
  window, nothing acted under a revoked grant, every hop of a journey covered).
  The verifier checks these too. See [Assertions](assertions.md).

A chain is **valid** only when every link recomputes (and matches any declared
`ref`) **and** every assertion holds.

## Documentation map

Start at the [Quickstart](quickstart.md) and read [Distribution](distribution.md)
to understand which surface gives you what. Then reach for the reference you need.

**Getting started**
- **[Quickstart](quickstart.md)**: install, verify the golden chain, verify a passport.
- **[Distribution](distribution.md)**: the independent PyPI pieces vs the complete suite.

**The verifier + passport** (the `algovoi-keystone-compose` package)
- **[CLI reference](cli.md)**: every `keystone-compose` command, flag, exit code.
- **[Chains](chains.md)**: the chain file format, `@name` binding, references, the cap.
- **[Aliasing](aliasing.md)**: map your own field names and variables into a chain via `@name`.
- **[Assertions](assertions.md)**: `scope_subset`, `field_equals`, `time_within`, `not_revoked`, `journey`, plus the scope-containment rules.
- **[Passports](passport.md)**: present, inspect and verify agent passports.
- **[Python API](python-api.md)**: every public function and dataclass with signatures.
- **[Conformance](conformance.md)**: Python and Node byte-parity, the self-test, the vectors.

**The keystone framework** (the complete suite)
- **[The `keystone` CLI](keystone-cli.md)**: `info` / `doctor` / `new` / `test` / `validate` / `journal` / `publish`.
- **[`keystone_ref` and connectors](keystone-ref.md)**: the primitive, the connector model, conformance helpers.
- **[Execution and decision refs](execution-ref.md)**: the `execution_ref` / `decision_ref` / `trust_query_ref` builders.
- **[The Journal and Runtime](journal.md)**: the tamper-evident SQLite journal and gated runtime.
- **[The agent engine](agent.md)**: rules, triggers, behaviours, the Engine, guarding writes.
- **[The Control Panel](control-panel.md)**: discover and manage every installed keystone piece.
- **[Building a bolt-on](bolt-ons.md)**: scaffold, implement, test and publish your own connector, behaviour or stage.

## What this documentation covers

This is the documentation for the open keystone framework: the offline verifier
(`keystone-compose`), the [`keystone_ref` primitive and connectors](keystone-ref.md),
the [execution and decision ref builders](execution-ref.md), the tamper-evident
[Journal and Runtime](journal.md), and the [agent engine](agent.md). Every piece is
content-addressed and recomputable offline, built on the one primitive.

Two things sit outside it. **Issuing** agent credentials is an issuer-side concern,
not part of these client-side tools. And the **commercial** AlgoVoi payment-rails
product builds on this framework for production deployments. The framework itself
is open, so anyone can verify and compose on the same trust base, independently of
who issued or operated it.

## License

Apache-2.0. Copyright AlgoVoi. Keep the `NOTICE` attribution when you
redistribute. See [`LICENSE`](../LICENSE) and [`NOTICE`](../NOTICE).
