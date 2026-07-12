# CLI reference

The Python distribution installs the `keystone-compose` command. (The Node
distribution `@algovoi/keystone` installs a `keystone` command with the same
chain-verification behaviour.)

```
keystone-compose verify <chain.json>
keystone-compose passport verify  <credential> [flags]
keystone-compose passport inspect <credential>
keystone-compose passport present <credential> (--x402 | --a2a)
```

Any `<credential>` argument may be given literally, or as `@path` to read the
credential from a file.

## Exit codes

The same three codes apply to every command:

| Code | Meaning |
| --- | --- |
| `0` | Success: the chain composes, or the passport is active, or the input decoded. |
| `1` | Broken: a link mismatched, an assertion failed, or the passport is not active (expired, revoked, invalid, unverifiable). |
| `2` | Bad input or usage: file not found, malformed JSON, a structurally invalid chain, or an unknown flag. |

The distinction between `1` and `2` matters in automation: `1` is a well-formed
artifact that does not verify (a real negative result), `2` is a problem with the
input or the invocation.

## `keystone-compose verify <chain.json>`

Recompute every link in a keystone chain in order, enforce structural composition
through `@name` references, and check any semantic assertions.

Output has three parts:

1. The chain, as an arrow-joined list of link names.
2. One block per link: `OK` or `BAD`, the recomputed reference, and (for links
   that reference earlier links) `binds <names>`. A `BAD` link also prints the
   mismatch: `recomputed <x> != declared <y>`.
3. If the chain carries assertions, an `assertions:` block with `OK`/`BAD` per
   assertion.

The final line is the verdict:

- `KEYSTONE VALID: N/N links compose, recompute byte-for-byte, no issuer contact` (plus, if present, `+ <assertion kinds> hold`). If the last link aggregates the chain, a `capped:` line names the cap.
- `KEYSTONE BROKEN at <link>: <detail>` when a link mismatches.
- `KEYSTONE BROKEN: <assertion> failed -- <detail>` when the links recompute but an assertion does not hold.

See [Chains](chains.md) for the file format and [Assertions](assertions.md) for
the assertion kinds.

## `keystone-compose passport verify <credential> [flags]`

Verify an agent passport fully offline. Fail-closed: with no key resolved or no
revocation confirmed, the verdict is `unverifiable`, never a default trust.

| Flag | Argument | Effect |
| --- | --- | --- |
| `--key` | `KID=PK_B64` | Supply one issuer public key by key id. May be repeated. |
| `--keys` | `FILE` | Load a JSON object of `{kid: pk_b64}` issuer keys from a file. |
| `--issuer-url` | `URL` | Opt in to online resolution: fetch any missing key from the issuer's well-known document and, unless `--crl` is given, fetch the revocation list for the credential's kid. |
| `--crl` | `FILE` | Load a revocation list (JSON) from a file, for offline revocation checking. |
| `--scope` | `S` | Require the passport to carry scope `S`; fail if it does not. |
| `--agent` | `DID` | Pin the expected agent DID; fail if the passport's subject differs. |
| `--no-revocation-check` | (none) | Do not require a revocation confirmation. Use only when revocation is out of scope; it relaxes the fail-closed default. |

Keys can be supplied inline (`--key`), from a file (`--keys`), or resolved online
(`--issuer-url`). Resolution is additive: online resolution fills only what was
not supplied. If online resolution fails, verification proceeds with whatever was
supplied and prints a warning.

Output prints the decoded subject (agent DID, issuer, kid, scopes, spend bound,
expiry), then the verdict:

- `PASSPORT ACTIVE: signature verifies offline, not expired, not revoked (status=active).` (exit `0`).
- `PASSPORT NOT ACTIVE: status=<status> [<error_code>] <error_message>` (exit `1`), where `<status>` is one of `revoked`, `expired`, `invalid`, `unverifiable`.

See [Passports](passport.md) for the full verdict model.

## `keystone-compose passport inspect <credential>`

Decode a passport and print its payload as sorted JSON, followed by a comment line
with the algorithm and kid. **Nothing is verified**: the signature is not checked.
Use this to read a credential you do not yet trust. Exit `0` on a decodable
credential, `2` if it cannot be decoded.

## `keystone-compose passport present <credential> (--x402 | --a2a)`

Emit the wire form for presenting a passport on an outbound call:

- `--x402` prints the HTTP header map (the passport goes in an `X-Agent-Passport` header).
- `--a2a` prints `{"metadata": ...}` for an A2A message (keyed under the passport metadata field).

Exit `0` on success, `2` if neither mode is given.
