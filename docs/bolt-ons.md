# Building a bolt-on

A **bolt-on** is a small, publishable package that extends keystone. You scaffold
it, fill in one function, prove it emits correct references, and publish it. There
are three kinds, one per role in a chain:

| Kind | Role | Built on |
| --- | --- | --- |
| `connector` | bind each write on a client to its reference | [`keystone_ref` + connectors](keystone-ref.md) |
| `behaviour` | govern a stage with an `ALLOW` / `FLAG` / `BLOCK` verdict | [the agent engine](agent.md) |
| `stage` | build the reference for a stage of the chain | [`keystone_ref`](keystone-ref.md) |

The lifecycle is the same for all three: **scaffold, implement, test, publish.**

## 1. Scaffold

```bash
keystone new --kind connector my_store        # or --kind behaviour / --kind stage
```

This writes a ready-to-publish package under `my_store/python/`:

```
my_store/python/
  keystone_my_store/__init__.py       # the one function you implement
  tests/test_keystone_my_store.py     # a conformance test, already wired
  pyproject.toml                      # name keystone-my-store, deps declared
  README.md
  NOTICE
```

The package depends on `algovoi-keystone-connect` (and `algovoi-keystone-agent`
for behaviours). Nothing else.

## 2. Implement

Each kind gives you one thing to fill in. The scaffolds below are exactly what
`keystone new` produces.

### connector

Map each write method to `(action, scope_fn)`. The `scope_fn` reads the call to
produce the per-call scope; every call to a mapped method then emits a reference.

```python
from algovoi_keystone_connect import connector

keystone_my_store = connector("my_store", writes={
    "write":  ("write",  lambda call: str(call.kwargs.get("key", ""))),
    "delete": ("delete", lambda call: str(call.kwargs.get("key", ""))),
})
```

### behaviour

Set the **trigger** (when it wakes) and the **rule** (what verdict it returns).

```python
from algovoi_keystone_agent import behaviour, rule, trigger

my_guard = behaviour(
    "my_guard",
    on=trigger(stage="execution", where=lambda ev: ev.get("method") is not None),
    rule=rule("my_guard_rule", lambda ev: "ALLOW"),   # return ALLOW / FLAG / BLOCK
)
```

### stage

Build a reference over your payload, binding upstream via a `*_ref` field.

```python
from algovoi_keystone_connect import keystone_ref

def my_stage_ref(payload):
    return keystone_ref(payload)   # bind to upstream through a *_ref field in payload
```

## 3. Test

The scaffold ships a conformance test that proves the bolt-on emits correct,
recomputable references, using the matching helper (`check_connector`,
`check_behaviour`, or `check_ref_builder`). Run it:

```bash
keystone test        # or: pytest -q
```

A freshly scaffolded bolt-on passes as generated, so you always start from green
and edit down from there. Example (connector):

```python
from algovoi_keystone_connect import check_connector
from keystone_my_store import keystone_my_store

class Fake:
    def __init__(self): self.calls = []
    def write(self, **kw):  self.calls.append(("write", kw));  return {}
    def delete(self, **kw): self.calls.append(("delete", kw)); return {}

def test_connector_conforms():
    report = check_connector(keystone_my_store, Fake(), [
        ("write",  {"key": "k1"}),
        ("delete", {"key": "k1"}),
    ])
    assert report.ok, report
```

## 4. Publish

```bash
keystone publish            # packages the current directory, mirror-first
# or: keystone publish path/to/my_store/python
```

`keystone publish` builds the package and runs the pre-publish checks so it is
ready to distribute.

## Make it discoverable by the Control Panel

The scaffold does not declare a discovery entry point by default. To have the
[Control Panel](control-panel.md) surface your bolt-on, add one to its
`pyproject.toml`, in the group that matches what it does:

| Entry-point group | For a bolt-on that | Kinds |
| --- | --- | --- |
| `algovoi.keystone.steps` | **produces** a reference | `connector`, `stage` |
| `algovoi.keystone.integrations` | **consumes or verifies** references | verifiers, `behaviour` guards |

Point the entry point at a callable that returns the bolt-on's descriptor. Then:

```bash
keystone-verify keystone-my-store     # confirm the panel discovers it
```

Install the bolt-on and it appears in the panel; uninstall it and it disappears.
No catalogue to edit, no panel rebuild.
