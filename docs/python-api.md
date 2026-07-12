# Python API

Everything below is importable from the top-level `algovoi_keystone` package.

```python
from algovoi_keystone import (
    # keystone chains
    verify_chain, ChainResult, LinkResult, AssertionResult, scope_contains, KeystoneError,
    # agent passports
    verify_passport, PassportVerdict, DecodedPassport, decode_passport, PassportError,
    kid_for_key, load_credential, x402_headers, a2a_message_metadata,
    resolve_keys_from_wellknown, fetch_crl,
)
```

## Chains

### `verify_chain(chain: dict) -> ChainResult`

Recompute every link in order, enforce structural composition through `@name`
references, and evaluate any assertions. `chain` is the parsed chain document:
`{"chain": [...], "assertions": [...]}`. Raises `KeystoneError` if the chain is
structurally invalid (missing `chain` array, a link without a `name` or
`preimage`, a duplicate name, a forward or dangling `@name`, or an assertion that
references something not in the chain). Otherwise returns a `ChainResult`.

### `ChainResult`

| Member | Type | Meaning |
| --- | --- | --- |
| `links` | `list[LinkResult]` | One per link, in order. |
| `assertions` | `list[AssertionResult]` | One per assertion. |
| `cap` | `str \| None` | Name of the final link if it aggregates two or more prior links. |
| `cap_covers` | `list[str]` | The links the cap binds, in order. |
| `capped` | `bool` | True if a cap was detected and the cap link is `ok`. |
| `ok` | `bool` (property) | True only if there is at least one link, every link is `ok`, and every assertion is `ok`. |
| `first_broken` | `LinkResult \| None` (property) | The first link that did not recompute. |
| `failed_assertion` | `AssertionResult \| None` (property) | The first assertion that did not hold. |

### `LinkResult`

| Member | Type | Meaning |
| --- | --- | --- |
| `name` | `str` | The link name. |
| `computed` | `str` | The recomputed reference (`sha256:...`). |
| `expected` | `str \| None` | The declared `ref`, if any. |
| `ok` | `bool` | `expected is None` or `computed == expected`. |
| `binds` | `list[str]` | Names of upstream links this preimage referenced. |
| `detail` | `str` | Mismatch detail when not `ok`. |

### `AssertionResult`

| Member | Type | Meaning |
| --- | --- | --- |
| `kind` | `str` | One of `scope_subset`, `field_equals`, `time_within`, `not_revoked`, `journey`. |
| `ok` | `bool` | Whether the assertion held. |
| `detail` | `str` | Explanation when not `ok`. |

### `scope_contains(outer, inner) -> tuple[bool, str]`

The containment primitive behind the `scope_subset` assertion. Returns
`(contained, reason)`; `reason` is empty when contained, otherwise a short
explanation. Rules are documented in
[Assertions: the scope-containment model](assertions.md#the-scope-containment-model).

### `KeystoneError`

Raised for a structurally invalid chain (as distinct from a chain that simply does
not verify, which is a normal `ChainResult` with `ok == False`).

## Passports

### `verify_passport(credential, *, issuer_keys=None, crl=None, required_scope=None, expected_agent_did=None, require_revocation_check=True, now=None) -> PassportVerdict`

Verify a passport offline. Performs no network I/O. See [Passports](passport.md)
for the parameters and the verdict model.

### `decode_passport(credential: str) -> DecodedPassport`

Decode a credential without verifying it. Raises `PassportError` if it cannot be
decoded. `DecodedPassport` has `payload`, `alg`, `kid`, `sig`, and `signed_bytes`.

### `PassportVerdict`

Frozen dataclass returned by `verify_passport`. Fields: `verified`, `status`,
`agent_did`, `scopes`, `spend_limit_microusd`, `spend_window`, `issuer_did`,
`kid`, `issued_at`, `expires_at`, `error_code`, `error_message`. See
[Passports: the verdict](passport.md#the-verdict).

### Present and resolve helpers

| Function | Returns | Purpose |
| --- | --- | --- |
| `x402_headers(credential)` | `dict` | HTTP header map to present a passport on an x402 call. |
| `a2a_message_metadata(credential, metadata=None)` | `dict` | A2A message metadata carrying the passport. |
| `load_credential(source=None)` | `str` | Read a credential from a source, or the standard environment variable when `None`. |
| `kid_for_key(public_key: bytes)` | `str` | Compute the kid for a public key. |
| `resolve_keys_from_wellknown(issuer_url, *, fetch=None, timeout=6.0)` | `dict` | `{kid: public_key}` from an issuer well-known document. |
| `fetch_crl(issuer_url, kid, *, fetch=None, timeout=6.0)` | `dict` | The revocation document for a kid. |

Both resolver helpers accept an injectable `fetch` callable so you can supply keys
and revocation from a self-signed endpoint, a cache, or a test double, and keep
`verify_passport` itself offline.

### `PassportError`

Raised when a credential cannot be decoded or is structurally malformed.
