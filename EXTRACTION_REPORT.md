# Source coverage and extraction limits

This repository documents the Rapido wire behavior observed in Rapido-VPN source
commit `084ddc73ddd9b442ef56c5af0ddd80430140536e`. The extraction revision is
2026-10-23, the accepted wire header version is 1, and the specification is a
draft implementation snapshot. The source commit is provenance, not a claim that
current public nodes or released clients run those exact bytes.

## Coverage

The six [specification sections](README.md) cover frame and control codecs,
session establishment, admission, key schedules, protected data, phase changes,
endpoint rebinding, UDP and WSS. The source inventory in
[protocol.md](specification/protocol.md#evidence-inventory) identifies the
implementation paths used to describe this behavior. The three JSON files in
[examples](examples/README.md) are byte-preserved synthetic fixtures from the
source snapshot; they are data examples, not SDKs or valid managed-account
credentials.

This specification resolves several differences between historical descriptions
and the pinned implementation:

- The accepted header version is 1 and the control codec is BinV1, despite old
  "v0" and JSON wording.
- Kind 11, exact optional tails, the whole-frame encoder budget, idempotent
  controls and the stalled-rehandshake replacement rule are documented.
- Explicit HTTPS mode uses WSS; there is no specified automatic QUIC/TCP
  fallback or inter-server multi-hop frame format.

The full [discrepancy table](specification/errors.md#resolved-documentation-discrepancies)
gives the corresponding behavior and links to its details.

## Lifecycle and security findings

The pinned server has one expiry entry for the immediately previous schedule-2
key phase. A second rollover within the 30,000-ms overlap overwrites that entry
without deleting an older key or its replay state. Authenticated old-phase data
can pass the retained-key and replay boundary after that period; later policy
and egress checks still govern forwarding. The overlap is therefore not a
global retirement bound. See [phase lifecycle](specification/handshake.md#key-phases-and-rehandshake)
and [remaining limits](specification/errors.md#remaining-limits-and-concrete-unknowns).

For a source-eligible, decodable keepalive requesting the next schedule-2 phase,
the server installs that phase before checking the keepalive counter. A
duplicate or lower counter then rejects without ACK or TTL refresh, but the new
active phase can be used for subsequent server return data. The client still
requires a matching ACK before its own phase cutover. See
[keepalive lifecycle](specification/handshake.md#state-transitions-and-expiry)
and [error handling](specification/errors.md#error-handling). This description
does not imply rollback on rejection or specify every malformed-frame side effect.

Schedules 0 and 1 derive traffic keys solely from public handshake fields.
Schedule 2 uses a custom client-ephemeral/server-static X25519 construction and
does not provide forward secrecy against later compromise of the server static
private key. These conclusions follow from the pinned key computations and are
conditional on the selected schedule. The [security section](specification/security.md)
describes the exact computations and limits; no deployed security property is
certified by this source extraction.

## Evidence limits

The frame encodings, ordered transcripts, key labels, nonce and AAD rules, state
transitions and carrier framing provide a basis for independent implementation.
The synthetic fixtures cover selected positive and negative cases, including a
schedule-2 key derivation vector. They do not prove complete native-client parity
or interoperability with an account-backed public service.

Account enrollment, signed-catalog integration and complete managed-account
captures are outside this specification. No remote service, device or release
was tested for this extraction. The phase-retention and rejected-keepalive
effects above are source-derived traces, not running-server reproductions.
Deployment configuration and actual release compatibility must be established
separately. Further concrete limits are listed in
[errors.md](specification/errors.md#remaining-limits-and-concrete-unknowns).

The specification text and data-only examples are under
[CC BY 4.0](LICENSE), with scope detailed in [LICENSING.md](LICENSING.md).
The separate Rapido VPN application implementation is excluded.
