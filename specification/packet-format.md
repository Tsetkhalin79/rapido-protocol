# Binary frame and message formats

## Primitive encodings

All integers are unsigned big-endian. `U8`, `U16`, `U32`, `U64` name their bit
width. `S(x)` is `U16(UTF-8 byte length) || UTF-8(x)`, without a terminator.
Strings must be valid UTF-8. `V(x)` is `U8(element count) || U8 elements` in
offered order. `||` means byte concatenation. Length prefixes count bytes, not
Unicode characters. Payload parsers require exact consumption, including tails.

## Common header

| Offset | Bytes | Meaning |
| --- | --- | --- |
| 0 | 1 | version = 1 |
| 1 | 1 | message kind |
| 2 | 2 | logical payload length |
| 4 | 2 | flags |
| 6 | 2 | reserved; emitted as 0 |
| 8 | payload length | payload |
| following | variable | optional padding trailer |

Control flags are `0x0002` (BinV1) optionally OR `0x0001` (padding).
The codec mask is `0x0006`. Other codec selections are rejected. The Rust
control decoder masks these bits rather than rejecting all other flag bits;
senders must emit only the defined values. Reserved is read but not validated by
the Rust header decoder; send 0. DataPacket requires flags exactly 0.

The reference encoder caps the **whole frame**, including header and padding, at
1200 bytes; this is not 1200 bytes of usable plaintext. The generic Rust decoder
checks truncation and padding consistency but does not itself impose that cap.
Carrier-specific receive bounds still apply. Senders should use the 1200-byte
frame budget for interoperability. If actual frame length exceeds header plus
payload length, the padding flag is required; padded trailer bytes are ignored.
Padding bytes are outside the logical payload and are not additional messages.
The reference encoder pads only SessionInit/SessionReady, with 0..64 random bytes
and a configured 70 percent attempt probability, constrained by the frame budget.
Padding is optional and neither authenticated encryption nor fragmentation.

## Control payloads

The payload does not contain its message kind a second time.

| Kind | Name | Mandatory payload, in order |
| --- | --- | --- |
| 1 | Ping | `U64(nonce)` |
| 2 | Pong | `U64(nonce)` |
| 3 | NestedVpnPolicySnapshot | `S(policy_mode) U16(mtu) U16(mss_clamp) U16(port_count)` then that many `U16(port)` |
| 4 | NestedVpnPolicyRequest | `S(session_id)` |
| 5 | NestedVpnPolicyAck | `S(session_id)` |
| 6 | SessionInit | `S(session_id) U64(client_nonce) S(user_id) S(device_id) U8(preferred_codec) V(transform_profiles)` |
| 7 | SessionReady | `S(session_id) U64(server_nonce) U8(policy_required) U8(selected_codec) U8(selected_transform_profile)` |
| 9 | SessionKeepalive | `S(session_id) U16(key_phase) U64(client_counter)` |
| 10 | SessionKeepaliveAck | `S(session_id) U16(key_phase) U64(client_counter) U64(server_time_ms)` |
| 11 | SessionClose | `S(session_id)` |

`policy_required` is emitted as 0/1; the Rust decoder treats any nonzero byte as
true. Codec negotiation is 0 absent or 2 BinV1; other bytes reject. A selected
transform byte of 0 maps to absence/baseline, otherwise carries the selected id.
There is no session_id in PolicySnapshot: associate it with the outstanding
request on the connection. Do not multiplex ambiguous pending snapshots.

### SessionInit tails

The payload may end after the mandatory fields or at the end of a complete group:

1. Credential group: `S(device_credential_id) S(device_credential_proof_scheme) S(device_credential_proof)`.
2. Capability group: `V(handshake_profiles) V(traffic_key_schedules)`.
3. X25519 group: `S(noise_nk_client_ephemeral_public_key)`.

Later groups require all earlier groups, using empty strings/vectors when absent.
Empty credential/ephemeral strings decode as absence. Partial groups and bytes
after the last group reject. Account-backed admission requires real credential
fields even though the codec supports a legacy tail-free form.

### SessionReady tails

The same complete-prefix rule applies:

1. `U8(selected_handshake_profile) U8(selected_traffic_key_schedule)`.
2. `S(noise_nk_server_static_key_id) S(noise_nk_server_static_public_key) S(noise_nk_transcript_hash)`.
3. `U16(selected_key_phase)`.

Empty strings mean absence; omitted selected phase uses initial phase 0 in the
legacy path. The selection bytes are optional as a pair. Senders cannot append
a phase without the preceding groups, even if their strings are empty.

### Keepalive extension

After the mandatory keepalive fields, either no bytes remain or an extension
starts with `U8(extension_flags)`. Bits: 1 challenge present, 2 proof present,
4 authentication phase present. The subsequent order is `U16(auth_key_phase)`
if bit 4, 32 raw challenge bytes if bit 1, 32 raw proof bytes if bit 2.
Zero flags and bits outside 0..2 reject. Proof and authentication phase must be
present together; a challenge requires a proof. Hence valid nonempty forms are
flags 6 (phase/proof) and 7 (phase/challenge/proof).

A KeepaliveAck may append exactly 32 raw challenge bytes without an extension
flag; any other tail length rejects. A challenge-bearing ACK is a challenge
response, not proof that the endpoint or key phase has been accepted.

## Kind 8: DataPacket

Payload: `S(session_id) U16(key_phase) U64(packet_counter) ciphertext_and_tag`.
Session id length must be 1..128 bytes. The remaining bytes are the complete
ChaCha20-Poly1305 output: ciphertext followed by the 16-byte tag. The codec only
requires a nonempty protected payload; AEAD rejects shorter/invalid protected
data. Do not apply the control codec flag to kind 8.

Nonce: `U32(key_phase) || U64(packet_counter)` (12 bytes).
Associated data: UTF-8 literal `rapido-data-v1` followed by
`S(session_id) || U16(key_phase) || U64(packet_counter)`.
The common header is not part of this associated data. Plaintext is one complete
IPv4 or IPv6 packet, without Ethernet framing. The data envelope and identifiers
remain visible on bare UDP. See [security](security.md) for exact keys.

Frame length for an IP packet of length `P` and session id byte length `N` is
`8 + 2 + N + 2 + 8 + P + 16 = 36 + N + P`. Both the frame budget and negotiated
MTU must hold: `P <= min(mtu, 1200 - 36 - N)`. A long allowed session id does not
guarantee that every packet at the default 1100-byte MTU fits the encoder.
