# The Control Panel

`algovoi-keystone-control` is the Control Panel: the public entry point to the
keystone suite. It auto-discovers every keystone piece installed in your
environment and gives you a browser UI to inspect and configure them. It is an
**independent** package on PyPI, and it is how you find your way into the
[complete suite](distribution.md).

```bash
pip install algovoi-keystone-control
```

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

The Control Panel runs a local browser UI over your installed keystone pieces,
behind a token login. Start it, open the printed URL, and authenticate with the
admin token from your environment.

Because the panel reads only what is installed locally and discovers pieces by
entry point, it is the single place to see your whole keystone footprint at once,
and the entry point to the rest of the complete suite.

> Note: the Control Panel ships as compiled wheels built for a specific Python
> version. Install it into a matching interpreter (see the package's PyPI page for
> the supported versions).
