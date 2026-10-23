# Authentication, key schedules and limitations

## Security properties of this snapshot

The names below are wire/source identifiers, not claims of standardized Noise
conformance or independently audited security. Schedules 0 and 1 contain no
secret input to their traffic-key KDF: the fields and nonces are observable in
the bare UDP handshake. An observer of that exchange can compute both traffic
keys. Device signature verification authenticates an enrolled client to the
server but does not make these KDF inputs secret. WSS adds TLS protection to
its carrier; it does not change these inner derivations.

Schedule 2 uses client-ephemeral/server-static X25519 plus a custom transcript.
There is no server ephemeral public key or standard Noise message processing in
this implementation. A later compromise of the server static private key allows
derivation of recorded sessions from their public client ephemeral keys. Thus
the identifier `noise_nk_x25519_chachapoly_sha256` and names containing
`forward_secret` must not be interpreted as forward secrecy against that
compromise. The transcript hash is an unkeyed SHA-256 hash; receiving it alone
is not cryptographic server key confirmation. Successful protected traffic
demonstrates possession of the derived traffic key, subject to these limits.

These are conclusions from the pinned key computations, not observations of a
deployed service. Repository defaults disable schedule-1 and schedule-2 server
gates; Android/iOS/Windows source implements all three schedules. Source names
such as `production_handshake_v1` do not justify a production-security claim.

## Common notation

Use [packet primitive encodings](packet-format.md). All literal labels are UTF-8
with no terminating NUL. `F(name,value)` is the UTF-8 byte sequence:
`name + ':' + decimal(UTF-8 byte length of value) + ':' + value + '\n'`.
The final newline is a single LF byte. Numeric values use unsigned base-10
without padding or sign. Lists use their original order as comma-separated
decimal ids, without spaces; the empty list is an empty string. Missing strings
are empty. Missing transform selection is numeric 0. Base64url uses `-`/`_`
and no `=` padding. X25519 inputs/public keys/shared secret are 32 raw bytes;
the protocol carries public keys as base64url strings.

Every traffic key is 32 bytes from HKDF-SHA256. Equivalently for this output
length: `PRK = HMAC-SHA256(salt, IKM)` and
`key = HMAC-SHA256(PRK, info || 0x01)`. There is no additional key truncation,
implicit string delimiter, hex conversion or terminating NUL.

## Enrolled device proof

Credential scheme is `rapido-device-p256-v1`; public-key format is
`p256-x963-base64url` (uncompressed SEC1 P-256 point encoded base64url).
Proof scheme is `rapido-device-p256-sha256-v1`. Sign the following transcript
with ECDSA P-256/SHA-256. Encode the signature as base64url; the server accepts
ASN.1 DER and, as a fallback, raw fixed-width `r || s` (32 bytes each).

Start with literal `rapido-device-credential-proof-v1\n`, then append `F` fields
in this exact order:

1. `session_id`, `client_nonce`, `user_id`, `device_id`, `device_credential_id`.
2. `proof_scheme` (the proof scheme above), `preferred_codec` (`BinV1`, including absent codec).
3. `transform_profiles` (CSV).
4. Only if either capability list is nonempty: `handshake_profiles`, `traffic_key_schedules` (both CSV).
5. Only if the ephemeral string is present and nonblank: `noise_nk_client_ephemeral_public_key`.

The server looks up the enrolled key by the supplied user/device/credential
identity and verifies this proof before SessionReady for managed users. Account,
device activity, entitlement, credential and revocation policy may reject
admission. This specification provides no alternate enrollment bypass. The
development unmanaged-identity exception is configuration-dependent and is not
a compatible way to access a managed account.

## Schedule 0

`IKM = S(session_id) || S(user_id) || S(device_id) || U64(client_nonce) || U64(server_nonce) || U16(key_phase)`.

Salt: `rapido-v0-bootstrap-data-plane`.

| Direction | HKDF info |
| --- | --- |
| client to server | `rapido c2s data traffic key v1` |
| server to client | `rapido s2c data traffic key v1` |

## Schedule 1

`IKM = S(session_id) || S(user_id) || S(device_id) || S(device_credential_id) || U64(client_nonce) || U64(server_nonce) || U16(key_phase) || U8(selected_transform_profile) || U8(selected_handshake_profile) || U8(selected_traffic_key_schedule)`.

Salt: `rapido-production-handshake-v1-data-plane`.

| Direction | HKDF info |
| --- | --- |
| client to server | `rapido production c2s data traffic key v1` |
| server to client | `rapido production s2c data traffic key v1` |

The credential identifier is not a shared secret; the ECDSA proof is not an IKM
field. Do not silently add it to the KDF.

## Schedule 2

Compute `shared_secret = X25519(client_ephemeral_private, server_static_public)`
on the client and `X25519(server_static_private, client_ephemeral_public)` on the
server. Reject an all-zero result. The client must have trusted the expected
server key id/public key through provisioning, not merely read them from the
same unauthenticated SessionReady it is checking.

The reference client trust channels are an authenticated account/device profile
bound to the selected server id, or a verified signed catalog with a configured
signing-key trust store. Repository defaults provide no such trusted key. Generic
provisioning of a pinned identity is sufficient to implement this wire exchange;
interoperation with the proprietary enrollment/catalog service is a separate
integration surface.

### Transcript hash

Start with `rapido-noise-nk-x25519-chachapoly-sha256-v1\n`. Append `F` fields:

1. `session_id`, `client_nonce`, `server_nonce`, `user_id`, `device_id`, `device_credential_id`.
2. `preferred_codec` (`BinV1`), `selected_transform_profile`.
3. `handshake_profiles`, `traffic_key_schedules` (CSV, both always included here).
4. `selected_handshake_profile`, `selected_traffic_key_schedule` (both 2).
5. `noise_nk_client_ephemeral_public_key`, `noise_nk_server_static_key_id`, `noise_nk_server_static_public_key`.

Hash the transcript bytes with SHA-256 and base64url-encode the 32-byte digest.
The client recomputes and compares this `noise_nk_transcript_hash`, the trusted
key id and the trusted public key. The selected phase is not a transcript field;
it is a separate field in the KDF below.

### Directional keys

Start IKM with the 32 raw shared-secret bytes, then append `F` fields:

1. `direction` (`client-to-server` or `server-to-client`).
2. `session_id`, `user_id`, `device_id`, `device_credential_id`.
3. `client_nonce`, `server_nonce`, `key_phase`.
4. `selected_transform_profile`, `selected_handshake_profile`, `selected_traffic_key_schedule`.
5. `noise_nk_transcript_hash` (the base64url text, not decoded hash bytes).

Salt: `rapido-noise-nk-x25519-chachapoly-sha256-data-plane`.
Info: `rapido-data-plane-chacha20poly1305` in both directions; direction is in IKM.

The existing [fixture](../examples/forward_secret_interop_fixtures.json) pins
these inputs, shared secret, transcript and directional keys. It uses deliberately
synthetic private bytes and a placeholder device proof; it is not an enrollment
credential or a complete valid managed-account handshake.

## Data protection and replay

Use ChaCha20-Poly1305 with the directional key, nonce and AAD specified in
[packet format](packet-format.md). AES-GCM and BLAKE2s are not negotiated
alternatives in this snapshot. Keep counters separate by session, direction and
key phase. Never reuse a key/nonce pair for a different plaintext. No wrapping
counter or phase transition is defined.

The server uses a 64-packet replay window per phase: seen counters and counters
at least 64 behind the highest accepted counter reject. Replay eligibility is
checked before decrypt; only successful AEAD authentication commits the counter.
Subsequent plaintext/policy/egress rejection does not authorize reuse of that
counter. Data decrypt alone is not proof of successful forwarding.

## Endpoint-rebind proof

Let `K` be the installed client-to-server traffic key for `auth_key_phase`.
`A = HMAC-SHA256(K, 'rapido-endpoint-rebind-auth-key-v1')`.

`proof = HMAC-SHA256(A, 'rapido-endpoint-rebind-proof-v1' || S(session_id) || U16(auth_key_phase) || U16(requested_key_phase) || U64(client_counter) || U16(challenge_length) || challenge)`.

The initial request uses an empty challenge, the challenge response uses the
32 raw bytes returned by the server. These proofs inherit the security of the
selected traffic-key schedule. The ordinary source-bound keepalive and close
frames have no whole-frame MAC; see [lifecycle](handshake.md).
