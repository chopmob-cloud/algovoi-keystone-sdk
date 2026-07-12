# The Journal and Runtime

`algovoi-keystone-runtime` is the durable, tamper-evident side of keystone: a
SQLite-backed **Journal** that stores keystone records in a hash chain you can
verify, and a **Runtime** that binds a decision to a journal and gates writes
through behaviours.

Part of the complete suite (see [Distribution](distribution.md)).

## Journal

### `Journal(path=":memory:", *, clock_ms=None)`

A durable, queryable store of keystone records, backed by SQLite. Use a file path
to persist; the default is in-memory.

```python
from algovoi_keystone_runtime import Journal

j = Journal("runtime.db")
j.append(record)          # append a self-describing keystone record
report = j.verify()       # every record recomputes AND the hash chain is intact
print(j.count())
j.close()
```

### Methods

| Method | Purpose |
| --- | --- |
| `append(record)` | Append a self-describing record; returns the new journal position. |
| `verify() -> Report` | Every record recomputes from its fields and the journal hash chain is intact (tamper-evident). |
| `records()` | The stored records. |
| `export()` | Every stored record as a plain list, for external offline verification. |
| `chain(decision_ref)` | Every record recorded under a decision, in order. |
| `by_decision(decision_ref)` / `by_kind(kind)` / `by_stage(stage)` | Filtered queries. |
| `head()` | The current chain head. |
| `count()` | Number of records. |
| `close()` | Close the underlying store. |

`append` accepts a self-describing record whose own reference recomputes from its
fields (for example an `execution_ref`, `behaviour_ref`, or `stage_ref` record).

### Tamper-evidence

`verify()` returns a `Report` that fails if any stored record no longer recomputes
or the hash chain linking records has been broken (a record altered, deleted, or
reordered). The same check is available from the CLI:

```bash
keystone journal runtime.db --show
```

## Runtime

### `Runtime(journal, *, decision_ref, clock_ms=None)`

Binds a decision to a Journal: it records each stage of the chain and can gate a
connect client's writes through [behaviours](agent.md).

```python
from algovoi_keystone_runtime import Runtime

rt = Runtime(journal, decision_ref="sha256:...")
rt.record_stage("passport", passport_payload)     # journal a stage of the chain
guarded = rt.bind(connect_client, behaviours=[my_behaviour])   # gate writes
```

| Method | Purpose |
| --- | --- |
| `record_stage(stage, payload)` | Journal a stage of the chain (passport, mandate, decision, ...). |
| `bind(connect_client, behaviours=())` | Return a client that gates writes through `behaviours` and journals what passes. |

## `verify_record(record) -> bool`

A standalone check: true when a record's own reference recomputes from its fields.
The building block `verify()` applies to every record in the journal.

```python
from algovoi_keystone_runtime import verify_record
assert verify_record(record)
```
