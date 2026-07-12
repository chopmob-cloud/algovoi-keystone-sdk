# Passports

An AlgoVoi agent passport is a Falcon-1024 (FIPS 206) credential that binds an
agent DID to a set of scopes, an optional spend bound, and an expiry. This SDK is
the **client half**: it presents a passport on an outbound call and verifies a
presented passport fully offline. Minting passports is an issuer-side concern and
is not part of this SDK.

Verifying passports needs a FIPS 206 verifier, so install the extra:

```bash
pip install "algovoi-keystone-compose[passport]"
```

## Credential format

A credential is `base64url(envelope)`, where the envelope is
`{payload, alg, kid, sig}`:

- `payload` the claims object (agent DID, scopes, spend bound, issued/expiry times).
- `alg` the signature algorithm (`Falcon-1024`).
- `kid` the key id: `SHA-256(public_key)` truncated, hex.
- `sig` the signature over `RFC 8785 JCS(payload)`, base64url.

Because the signature is over the JCS canonical bytes of the payload, the same
canonicalization used for keystone references applies here too: a verified
passport payload has a well-defined keystone reference.

## Present a passport

Attach a passport to an outbound call. From the CLI:

```bash
keystone-compose passport present <credential> --x402   # -> HTTP header map
keystone-compose passport present <credential> --a2a    # -> A2A message metadata
```

From Python:

```python
from algovoi_keystone import x402_headers, a2a_message_metadata

headers = x402_headers(credential)                 # {"X-Agent-Passport": "<credential>"}
meta    = a2a_message_metadata(credential)         # {"<passport metadata key>": "<credential>"}
```

`load_credential(source=None)` reads a credential from an explicit source or from
the standard agent-passport environment variable when `source` is `None`.

## Inspect a passport

Decode without trusting. The signature is **not** checked.

```bash
keystone-compose passport inspect <credential>
```

```python
from algovoi_keystone import decode_passport

dec = decode_passport(credential)   # DecodedPassport(payload, alg, kid, sig, signed_bytes)
print(dec.payload, dec.alg, dec.kid)
```

## Verify a passport

Offline verification with a fail-closed verdict.

```python
from algovoi_keystone import verify_passport

verdict = verify_passport(
    credential,
    issuer_keys={kid: public_key},     # mapping of kid -> Falcon public key (bytes or base64)
    crl=None,                          # revocation data, or None
    required_scope="pay:invoice",      # optional: require this scope
    expected_agent_did="did:key:...",  # optional: pin the subject
    require_revocation_check=True,      # default: fail closed if revocation unconfirmed
    now=None,                          # default: datetime.now(timezone.utc); injectable
)
```

`verify_passport` performs **no network I/O**. Resolve the issuer key and CRL up
front (see [Resolving keys and revocation](#resolving-keys-and-revocation)) and
pass them in.

### The verdict

`verify_passport` returns a frozen `PassportVerdict`:

| Field | Meaning |
| --- | --- |
| `verified` | `True` only when the passport is active. |
| `status` | One of `active`, `revoked`, `expired`, `invalid`, `unverifiable`. |
| `agent_did` | The subject agent DID. |
| `scopes` | Tuple of granted scopes. |
| `spend_limit_microusd`, `spend_window` | The spend bound, if any. |
| `issuer_did`, `kid` | The issuer identity and key id. |
| `issued_at`, `expires_at` | Issuance and expiry timestamps. |
| `error_code`, `error_message` | Set when not active, explaining why. |

### Status meanings

| Status | When |
| --- | --- |
| `active` | Signature verifies, not expired, not revoked, and any `required_scope` / `expected_agent_did` constraints hold. |
| `expired` | Well-formed and authentic, but past its expiry. |
| `revoked` | Present in the revocation data. |
| `invalid` | Malformed, or the signature does not verify against the resolved key. |
| `unverifiable` | No key could be resolved, or (with `require_revocation_check=True`) revocation could not be confirmed. Never treated as trust. |

The `kid` is bound to the key: it is `SHA-256(public_key)` truncated. A swapped key
does not match the credential's kid, so it cannot silently validate.

## Resolving keys and revocation

Because verification is offline, fetch the trust material first. Two helpers do the
network step, each accepting an injectable `fetch` callable (useful for
self-signed endpoints or fully offline testing):

```python
from algovoi_keystone import resolve_keys_from_wellknown, fetch_crl, kid_for_key

keys = resolve_keys_from_wellknown("https://issuer.example")   # {kid: public_key}
crl  = fetch_crl("https://issuer.example", kid)                # revocation document
kid  = kid_for_key(public_key_bytes)                           # compute a kid yourself
```

From the CLI, `--issuer-url` opts into the same resolution, filling only what was
not supplied by `--key` / `--keys` / `--crl`:

```bash
keystone-compose passport verify <credential> --issuer-url https://issuer.example --scope pay:invoice
```

If online resolution fails, the CLI proceeds with whatever was supplied and prints
a warning; the verdict stays fail-closed.

## Passports and chains together

A passport carries **authenticity** (a Falcon signature over the claims). A
keystone chain carries **composition** (content-addressed links that bind). They
meet on one primitive: the JCS canonical bytes of the passport payload have a
keystone reference, so a verified passport can be dropped into a chain as a
content-addressed link. Authenticity and composition, one trust base.
