# The connector library

You do not have to write a connector for common systems. Keystone ships a library
of ready-made connectors that bind writes on a real client (a database, a queue, a
bucket) to a keystone decision, so every write emits a content-addressed execution
reference you can later recompute and tamper-check.

Each connector is `algovoi-keystone-<system>`, part of the complete suite (see
[Distribution](distribution.md)). Every one depends only on `algovoi-execution-ref`,
and is discoverable by the [Control Panel](control-panel.md).

## What ships

| Category | Connectors |
| --- | --- |
| Databases and ORM | `sqlalchemy`, `mongo`, `dynamodb`, `odbc`, `elasticsearch` |
| Cache and key-value | `redis` |
| Object storage | `s3`, `gcs`, `azure-blob` |
| Message queues and streaming | `kafka`, `nats`, `amqp`, `sqs`, `sns`, `pubsub`, `celery`, `azure-servicebus` |
| RPC and HTTP | `grpc`, `httpx`, `asgi` |
| Vector databases | `pinecone`, `qdrant` |
| Data lineage | `openlineage` |

Building on a system that is not here? Scaffold your own with
[`keystone new --kind connector`](bolt-ons.md).

## The shape they share

Every connector exposes the same three things, named for its system:

1. **A wrapper** that binds a client to a `decision_ref`, so each write emits a reference.
2. **A ref builder** (`<system>_execution_ref(...)`) to build one reference by hand.
3. **`tamper_detected(...)`** to check a claimed reference against the write's fields.

### Worked example: Redis

```python
from algovoi_keystone_redis import keystone_redis, redis_execution_ref, tamper_detected

# 1. wrap a redis-py client, bound to the decision that authorized these writes
client = keystone_redis(real_redis, decision_ref="sha256:...")
client.set("user:1", "...")     # performs the write AND emits an execution ref

# 2. or build the reference for one write directly
ref = redis_execution_ref(
    decision_ref="sha256:...", command="SET", key="user:1",
    outcome="COMMITTED", executed_at_ms=1716460800000,
)

# 3. later, verify a claimed reference against the write it describes
tamper_detected(ref, decision_ref="sha256:...", command="SET", key="user:1",
                outcome="COMMITTED", executed_at_ms=1716460800000)   # False = intact
# change any field (e.g. key -> "user:2") and tamper_detected returns True
```

The wrapper takes `decision_ref` (the keystone this write acts under), an optional
`clock_ms` callable, and an optional `on_execution` callback that receives each
emitted record. The `decision_ref` must be a valid keystone reference
(`sha256:` + 64 lowercase hex).

### The same shape across systems

Only the write-identifying fields change per system:

| Connector | Ref builder | Write-identifying fields |
| --- | --- | --- |
| `redis` | `redis_execution_ref` | `command`, `key` |
| `s3` | `s3_execution_ref` | `bucket`, `key`, `action` (default `put`) |
| `sqlalchemy` | `orm_execution_ref` | `operation`, `table` (via `bind_session` after-flush listener) |

All builders also take `decision_ref`, `outcome`
(`COMMITTED` / `SKIPPED` / `FAILED` / `REVERSED`), and `executed_at_ms`, and all
return `"sha256:" + SHA-256(RFC 8785 JCS(preimage))` so the reference composes with
the rest of a [chain](chains.md) and can be [journaled](journal.md).

### SQLAlchemy: bind a whole session

The ORM connector registers an `after_flush` listener, so every flushed change is
bound automatically:

```python
from algovoi_keystone_sqlalchemy import bind_session, orm_execution_ref

bind_session(session, decision_ref="sha256:...")   # every flush now emits a ref
```

## Discovery

Each connector declares a keystone entry point, so once installed it appears in the
[Control Panel](control-panel.md) with no configuration. Confirm one with:

```bash
keystone-verify algovoi-keystone-redis
```
