# The `keystone` CLI (umbrella)

The complete suite installs the `keystone` command: the SDK's front door for
inspecting your environment, scaffolding a bolt-on, and validating and publishing
it. It is distinct from `keystone-compose` (the standalone chain verifier); this
is the umbrella that ties the suite together.

```
keystone {info,doctor,new,test,validate,journal,publish}
```

Part of the complete suite (see [Distribution](distribution.md)).

## `keystone info`

Show the installed keystone pieces and the one primitive they share.

```
$ keystone info
Keystone SDK -- installed pieces:
  algovoi-keystone           0.1.1
  algovoi-keystone-connect   0.1.0
  algovoi-keystone-agent     0.1.0
  algovoi-execution-ref      0.1.1
  rfc8785                    0.1.4

One primitive:
  keystone_ref(payload) == "sha256:" + SHA-256(RFC 8785 JCS(payload))
```

Use it to confirm what you have and that the primitive is the one you expect.

## `keystone doctor`

Check the environment: that the primitive computes correctly and the installed
pieces are consistent. Run it first when something is not lining up.

## `keystone new [--kind {connector,behaviour,stage}] <name>`

Scaffold a publishable bolt-on. The `--kind` picks the template:

- `connector`: a data-plane connector (wraps a client so its writes emit refs).
- `behaviour`: an agentic rule/trigger/action unit (see [the agent](agent.md)).
- `stage`: a ref builder for a stage in the chain.

The scaffold includes tests and the packaging so it is publishable from the start.

## `keystone test`

Run the bolt-on's tests. The scaffold ships conformance-style tests
(`check_connector`, `check_behaviour`, `check_ref_builder`) so a bolt-on proves it
emits correct refs before it ships.

## `keystone validate <file>`

Verify emitted refs offline. Give it a file of keystone records; it recomputes
each ref from its fields and reports whether they hold, with no issuer contact.
This is the record-level check (each ref recomputes); for full chain composition
use [`keystone-compose verify`](cli.md).

## `keystone journal [--show] <db>`

Verify a runtime [Journal](journal.md) database: every record recomputes and the
journal's hash chain is intact (tamper-evident). Add `--show` to list the entries.

```
keystone journal ./runtime.db --show
```

## `keystone publish [<path>]`

Build and check a bolt-on, mirror-first. It packages the bolt-on at `path` (or the
current directory) and runs the pre-publish checks so it is ready to distribute.
