# Distribution: independent pieces vs the complete suite

Keystone is delivered on two surfaces by design. Knowing which is which tells you
what to install and where each piece comes from.

## Two surfaces

**1. PyPI hosts the independent variations.** These are standalone packages that
each work on their own, with nothing else installed. Pick the one that does the
job you need:

| Package | Install | What it gives you |
| --- | --- | --- |
| `algovoi-keystone-compose` | `pip install algovoi-keystone-compose` | The offline **chain verifier** + agent-passport verify (this repo's docs) |
| `algovoi-execution-ref` | `pip install algovoi-execution-ref` | The `execution_ref` / `decision_ref` / `trust_query_ref` builders + CLI |
| `algovoi-keystone-control` | `pip install algovoi-keystone-control` | The **Control Panel** that discovers and manages installed keystone pieces |

Each is self-contained: install it, use it, done. This is the right surface when
you want one capability and nothing more.

**2. The complete suite is the integrated framework.** The full keystone SDK is
the umbrella plus its pieces working together:

| Piece | Role |
| --- | --- |
| `algovoi-keystone` (umbrella `keystone` CLI) | `info` / `doctor` / `new` / `test` / `validate` / `journal` / `publish` |
| `algovoi-keystone-connect` | the `keystone_ref` primitive + the connector model |
| `algovoi-keystone-runtime` | the tamper-evident **Journal** and gated **Runtime** |
| `algovoi-execution-ref` | the execution / decision / trust-query ref builders |
| `algovoi-keystone-agent` | the rule / trigger / behaviour **Engine** |

The complete suite is distributed as the **AlgoVoi Keystone distribution**. The
public entry point is the **Control Panel**: install it, and it discovers and
surfaces every keystone piece you have, and points you at the rest of the suite.
See [the Control Panel](control-panel.md) and the hosted docs at
`docs.algovoi.co.uk/keystone`.

## How to choose

- **You need one thing** (verify a chain, build an `execution_ref`): take the
  matching **independent** package from PyPI. Nothing else required.
- **You are building on keystone** (composing stages, journaling a runtime,
  deploying behaviours, scaffolding and publishing your own bolt-on): use the
  **complete suite** through the Control Panel, so the pieces are present and
  discover each other.

## One primitive underneath both

Every piece on either surface is built on the same single primitive:

```
keystone_ref(payload) == "sha256:" + SHA-256(RFC 8785 JCS(payload))
```

That is why the independent packages and the complete suite interoperate: a ref
built by one is byte-identical to the same ref built by another. See
[the keystone_ref primitive](keystone-ref.md).
