# The Control Panel

`algovoi-keystone-control` is the Control Panel: the public entry point to the
keystone suite. It auto-discovers every keystone piece installed in your
environment and gives you a browser UI to inspect and configure them. It is an
**independent** package on PyPI, and it is how you find your way into the
[complete suite](distribution.md).

```bash
pip install algovoi-keystone-control
```

## Start here: the panel-first workflow

The Control Panel is the recommended way into keystone. The path a developer takes:

1. **Install and start the panel:**
   ```bash
   pip install algovoi-keystone-control
   algv-keystone        # serves the panel; open the printed URL, log in with the admin token
   ```
2. **Install the keystone pieces you want to work with** (the panel auto-detects
   them, no configuration). For example:
   ```bash
   pip install algovoi-execution-ref        # an independent piece from PyPI
   # and the rest of the suite (connect, runtime, agent, connectors) per Distribution
   ```
3. **Confirm each piece is discovered:**
   ```bash
   keystone-verify algovoi-execution-ref
   ```
   Installed pieces appear in the panel; uninstalled ones disappear.
4. **Work with the SDK.** From here you use the pieces directly:
   [verify a chain](cli.md), [build and journal a runtime](journal.md), [deploy a
   behaviour](agent.md), [wire a connector](connectors.md), or
   [scaffold your own bolt-on](bolt-ons.md). See [Distribution](distribution.md)
   for which surface each piece comes from.

## Discovery: no catalogue to edit

The panel discovers keystone pieces through Python entry points. A package becomes
visible to the panel simply by declaring one; there is no central list to update
and no panel rebuild.

| Entry-point group | Meaning | Shown as |
| --- | --- | --- |
| `algovoi.keystone.steps` | the package **produces** a ref (a stage in the chain) | a step |
| `algovoi.keystone.integrations` | the package **consumes or verifies** refs | an integration |

A package registers a callable under the group; the panel calls it to get a
descriptor (`name`, `role`, `posture`, `package`) and renders a row for it. Install
a keystone piece and it appears; uninstall it and it disappears.

Example: a package exposes itself to the panel by declaring, in its packaging
metadata, an entry point in one of those groups pointing at a function that returns
its descriptor. Nothing else is required.

## Confirm a package is discovered

```bash
keystone-verify <package-name>
```

Reports whether the panel discovers the package and what descriptor it exposes.
Use it after adding an entry point to a bolt-on to confirm it will show up.

## Serve the panel

```bash
algv-keystone
```

The `algv-keystone` command runs the Control Panel as a local browser UI (a
FastAPI / uvicorn app) over your installed keystone pieces, behind a token login.
Open the printed URL and authenticate with the admin token from your environment.

Because the panel reads only what is installed locally and discovers pieces by
entry point, it is the single place to see your whole keystone footprint at once,
and the entry point to the rest of the complete suite.

> Note: the Control Panel ships as compiled wheels built for a specific Python
> version. Install it into a matching interpreter (see the package's PyPI page for
> the supported versions).
