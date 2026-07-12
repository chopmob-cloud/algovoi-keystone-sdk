# The `keystone_ref` primitive and connectors

`algovoi-keystone-connect` is the primitive every other piece is built on: one
function that turns any JSON value into a content-addressed reference, plus a
declarative way to make a client's writes emit those references (a connector).

Part of the complete suite (see [Distribution](distribution.md)).

## `keystone_ref(payload) -> str`

```python
from algovoi_keystone_connect import keystone_ref

ref = keystone_ref({"agent_id": "agent-001", "scope": "payments"})
# "sha256:" + SHA-256(RFC 8785 JCS(payload))
```

The reference is `"sha256:" + SHA-256(RFC 8785 JCS(payload))`. RFC 8785 (JCS)
canonicalization fixes a single deterministic byte string for the payload, so the
same value produces the same reference in any language or runtime. This is the one
primitive named by `keystone info`, and the basis of every stage reference,
journal record, and chain link across the suite.

## Connectors: make writes emit refs

A connector wraps a client so that each write it performs also emits a
content-addressed reference, without you hand-building the ref each time.

### `connector(prefix, *, writes) -> Callable`

```python
from algovoi_keystone_connect import connector

# declare which client methods write, and the stage each emits
make = connector("mypay", writes={"charge": "execution", "refund": "execution"})
client = make(real_client)         # wrap your real client
client.charge(amount=100)          # performs the write AND emits an execution ref
```

`writes` is a declarative spec mapping method names to the stage they emit. The
returned factory wraps a real client; calls to the declared methods pass through
and produce the reference for that stage.

## Stand-ins for upstream stages

### `synth_ref(stage="decision", **fields) -> str`

A deterministic, content-addressed stand-in for an upstream stage, so you can
build and test a downstream piece without wiring the whole chain first.

```python
from algovoi_keystone_connect import synth_ref
decision = synth_ref("decision", verdict="ALLOW")   # a stable placeholder ref
```

## Conformance helpers

Prove a bolt-on emits correct references before it ships:

| Function | Purpose |
| --- | --- |
| `check_connector(factory, client, calls, *, decision_ref=None) -> Report` | Conformance-test a connector at the execution stage. |
| `check_ref_builder(build, payload, *, prev_field=None) -> Report` | Conformance-test any stage's ref builder (passport, mandate, decision, ...). |

Both return a `Report` (a list of named `(passed, detail)` checks). This is what
`keystone test` runs against a scaffolded bolt-on.

```python
from algovoi_keystone_connect import check_ref_builder
report = check_ref_builder(my_build, {"field": "value"})
assert report.ok
```
