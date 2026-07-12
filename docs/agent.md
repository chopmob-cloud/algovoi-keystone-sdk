# The agent engine

`algovoi-keystone-agent` is the behavioural layer: declare **rules**, bind them to
**triggers**, package them as **behaviours**, and deploy them in an **Engine** that
fires on a stream of keystone events and can guard a client's writes. Every firing
is itself a content-addressed keystone record.

Part of the complete suite (see [Distribution](distribution.md)).

## The model

An **event** is a keystone record plus the chain stage it belongs to. A
**behaviour** is: when a **trigger** matches an event, evaluate a **rule** and take
an **action**. An **Engine** deploys a set of behaviours against events.

### Rules

```python
from algovoi_keystone_agent import rule

# a pure predicate returning a verdict: "ALLOW" / "FLAG" / "BLOCK"
high_value = rule("high-value", lambda ev: "BLOCK" if ev.amount > 1000 else "ALLOW")
```

`rule(name, predicate)` builds a `Rule`; `predicate(event)` returns a verdict.

### Triggers

```python
from algovoi_keystone_agent import trigger

on_execution = trigger(stage="execution")               # match a stage
on_charge = trigger(stage="execution", where=lambda ev: ev.action_type == "payment")
```

`trigger(stage=None, where=None)` matches an event when its stage equals `stage`
(`None` matches any stage) and the optional `where(event)` predicate holds.

### Behaviours

```python
from algovoi_keystone_agent import behaviour

bhv = behaviour("block-high-value", on=on_execution, rule=high_value, action=None)
```

`behaviour(name, *, on, rule, action=None)` is a deployable unit: when trigger `on`
matches, evaluate `rule`, and optionally run `action`.

## The Engine

### `Engine(behaviours, *, decision_ref, clock_ms=None, on_fire=None)`

Deploys a set of behaviours against a stream of events.

```python
from algovoi_keystone_agent import Engine, Denied

engine = Engine([bhv], decision_ref="sha256:...")
result = engine.dispatch(event)     # fire every matching behaviour; each firing = a behaviour_ref
verdict = engine.verdict(event)     # the strongest verdict across all that fire
```

| Method | Purpose |
| --- | --- |
| `dispatch(event)` | Fire every behaviour whose trigger matches; each firing produces a keystone record. |
| `verdict(event)` | The strongest verdict across all behaviours that fire on the event. |
| `guard(connect_client, *, stage="execution")` | Wrap a keystone-connect client so each write is evaluated first; a `BLOCK` raises `Denied`. |

### Guarding writes

```python
guarded = engine.guard(connect_client, stage="execution")
try:
    guarded.charge(amount=5000)      # evaluated by behaviours before it runs
except Denied as d:
    ...                              # a behaviour returned BLOCK; d carries the record
```

`Denied` is raised by a guarded write when a behaviour returns `BLOCK`, and carries
the keystone record explaining why.

## Testing behaviours

Prove a behaviour does what you intend before deploying it:

| Function | Purpose |
| --- | --- |
| `check_rule(rule_obj, cases)` | `cases = [(event, expected_verdict), ...]`; asserts the rule evaluates as expected. |
| `check_trigger(trig, matching, non_matching=())` | Asserts a trigger matches / does not match the given events. |
| `check_behaviour(bhv, events, *, decision_ref=None, expect=None, clock_ms=None)` | Deploy a behaviour in a throwaway Engine over events and assert the keystone result. |
| `synth_event(stage="execution", **fields)` | A content-addressed stand-in event for whatever precedes your behaviour. |

```python
from algovoi_keystone_agent import check_rule, synth_event
check_rule(high_value, [(synth_event(amount=2000), "BLOCK"), (synth_event(amount=10), "ALLOW")])
```

A scaffolded behaviour (`keystone new --kind behaviour`) ships these checks, and
`keystone test` runs them.
