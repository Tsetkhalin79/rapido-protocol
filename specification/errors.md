# Rejections, discrepancies and limits

## Error handling

There is no generic binary Error message kind in this snapshot. Most invalid UDP
input is dropped without a success response. A local log/metric reason or JSON
test fixture `expected_error` is not an error string transmitted over UDP.
WSS connection-level HTTP/close errors are described in [transports](transports.md).

| Rejection class | Examples | Result |
| --- | --- | --- |
| Framing | short header/payload, unsupported version/kind, trailing bytes without padding, invalid codec bits | no valid control/data processing |
| Binary payload | invalid UTF-8, malformed length/tail, unsupported codec negotiation, malformed keepalive extension | decode rejects before normal control/data dispatch |
| Admission | missing identity, invalid device signature, unavailable account lookup, inactive/revoked identity, inactive entitlement, node capacity policy | no successful bootstrap |
| Selection | unknown transform/profile/schedule, unavailable gated material, opted-in client downgrade | no usable selected session |
| Sequence | ACK before policy snapshot, request after Established, keepalive before establishment, pending-rehandshake conflict | no success response |
| Replay/lifetime | old init nonce, expired session/challenge, stale or duplicate data counter | reject; no successful liveness confirmation |
| Keepalive counter replay | duplicate/lower counter on a source-eligible keepalive requesting the next schedule-2 phase | reject without ACK or TTL refresh; the phase can already be installed |
| Data | missing phase key, bad AEAD, invalid IP lengths/MTU, wrong source/policy or unavailable egress | packet dropped; no forwarding guarantee |
| Recovery | bad rebind proof, unmatched challenge/generation, skipped/exhausted phase | no accepted endpoint/phase transition |

A single timeout does not distinguish rejection, congestion, packet loss, TLS
failure or an unreachable endpoint. Implementations must match the expected
response and state, not treat socket success or Ping as authenticated acceptance.
Local diagnostics may classify causes but are not a wire error vocabulary.
The keepalive counter check follows next-phase installation in the pinned server.
This specific rejection has a phase side effect: subsequent server return data
can use the newly active phase, while the client has no matching ACK authorizing
its own phase cutover. A rejection is not a general rollback guarantee; these
rows do not specify side effects for every malformed frame.

## Resolved documentation discrepancies

| Old statement | Pinned implementation and this specification |
| --- | --- |
| "v0" header/protocol terminology | Header version is 1; v0 is a historical baseline/schedule name. |
| Padding payload described as JSON | The accepted control codec is BinV1; JSON flags reject. |
| Noise IK-style profile, BLAKE2s/AES alternatives, ephemeral forward secrecy | Only the documented HKDF-SHA256/ChaCha20-Poly1305 schedules exist; schedule 2 is a custom single-DH construction. See security limitations. |
| Initial max payload 1200 | Reference encode cap applies to entire frame, including padding and envelope. Generic decode does not enforce that same bound. |
| Initial kind table ends at 10 | Kind 11 SessionClose and keepalive-rebind/selection tails are implemented. |
| Every duplicate transition rejects | PolicyRequest before establishment and ACK after establishment have explicit idempotent behavior. |
| UDP-first automatic QUIC/TCP fallback | Explicit HTTPS mode uses WSS; no generic automatic failover wire protocol or QUIC carrier is defined. |
| Any overlapping fresh-secret init rejects | A newer valid pending replacement is allowed after the implemented 5000-ms grace. |
| Every previous phase retires after a 30,000-ms overlap | Only the immediately previous phase has an expiry entry. A consecutive rollover can overwrite that entry, leaving older keys and replay state available until other session cleanup. See [key phases](handshake.md#key-phases-and-rehandshake). |
| Default unknown identities admitted unconditionally | Admission is layered and configurable; managed-account identity/credential/entitlement checks apply. |
| Proposed Entry/Exit and multi-hop architecture described alongside wire baseline | Those product targets do not define implemented inter-server framing in this snapshot. |

## Remaining limits and concrete unknowns

1. This documents a pinned local committed snapshot, not the exact binaries or
   configuration running on public nodes or installed clients. No remote service,
   device, release or network exchange was tested during extraction.
2. Codec, key schedules and carrier layout are specified for independent peer
   implementation. Proprietary account/enrollment/catalog APIs and their full
   signed-catalog canonicalization schema are not extracted here. A peer needs
   out-of-band enrolled user/device credentials and a trusted server identity
   where schedule 2 is selected. This repository alone cannot register a public
   service account.
3. Existing positive/negative fixtures cover selected cases, not every message
   tail, phase, source-rebind trace and platform combination. The schedule-2
   fixture has a placeholder proof. No existing complete captured, sanitized,
   cryptographically authenticated managed-account end-to-end transcript was
   established by this extraction. No independent interoperability certification
   is claimed.
4. Platform decoders and local retries have additional implementation-specific
   limits. The common complete-prefix forms and conservative frame budget are
   documented; equivalence on every malformed-input or Unicode-normalization
   edge across all native clients has not been exhaustively established. Use
   stable ASCII identity strings and the documented exact UTF-8 lengths.
5. Consecutive schedule-2 rollovers can leave an older phase's key and replay
   state without the advertised 30,000-ms retirement bound. A valid old-phase
   data frame may pass the retained-key/replay boundary; later policy and egress
   checks still govern delivery. This source-derived trace has not been
   reproduced against a running server, and no runtime remedy is included here.
6. Deployment-specific capacity/rate/TTL values and transport availability are
   not negotiated comprehensively on the wire. No universal retry interval or
   recovery success deadline can be inferred from the packet format.

For licensing scope, see [LICENSING.md](../LICENSING.md). These limits describe
the documented snapshot; they do not imply a runtime remedy for the identified
discrepancies.
