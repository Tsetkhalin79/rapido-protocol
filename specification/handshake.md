# Session establishment and lifecycle

## Establishment sequence

Provision a nonempty session id, user id, device id and enrolled credential for
a managed account. Native clients require identity rather than inventing a
development account. Client nonces are unsigned integers; a replacement under
the same session id must exceed the existing and pending client nonce. Server
nonces come from a monotonic process-local counter (initially 1), not a random
challenge. Do not rely on their unpredictability or persistence across restart.

1. Client sends SessionInit with supported transform/profile/schedule lists and
   the device proof described in [security](security.md).
2. Server checks identity/admission and selection and returns SessionReady.
3. Client verifies session id, codec and selection; for schedule 2 also verifies
   the pretrusted static key and recomputed transcript. An explicitly opted-in
   native schedule-2 client rejects lower or mixed profile/schedule selection.
4. Client sends NestedVpnPolicyRequest; server replies with PolicySnapshot.
5. Client applies the local routing/MTU policy and sends NestedVpnPolicyAck.
6. Server installs both directional keys and echoes NestedVpnPolicyAck. Wait for
   this echo before treating bootstrap as complete and sending protected data.

The source baseline policy is `preserve-secondary-routes`, MTU 1100, MSS clamp
1048, passthrough UDP ports `[500,4500]`. These are policy values, not new cipher
parameters or guarantees that every nested tunnel works. No address assignment,
DNS resolver list or route list is carried by this snapshot message.

## Selection

Reject unsupported ids. If both offered lists contain 2 and the schedule-2 server
gate is enabled, require nonempty device proof fields, valid client ephemeral
key and configured static identity; failure rejects rather than falling back
inside that selection. Otherwise, gate-enabled schedule 1 with both offered 1s
and proof fields selects 1/1. Otherwise choose 0/0 only if both lists are empty
or contain 0. If no pair qualifies, reject.

The lower-level selection function checks field presence; the outer managed
admission path verifies the actual signature. These are separate checks.
The server may select transform 1 only if offered and requested by its adaptive
override; baseline/absent selection is 0. Unknown transform offers reject.

## State transitions and expiry

States are AwaitingPolicyRequest, AwaitingPolicyAck, Established.
PolicyRequest is accepted in either first or second state (retransmission returns
the snapshot), but rejects once established. ACK before a snapshot rejects.
An ACK repeated after establishment is accepted, refreshes TTL and returns an
ACK without resetting the installed data keys. This is not a generic policy of
rejecting all duplicate control messages.

The default server session TTL is 86,400,000 ms. Expiry is exact at
`expires_at_ms <= now_ms`. Accepted policy progress, accepted keepalive and
accepted data activity refresh the configured lease. Deployment may override
TTL; it is not negotiated in a frame. Restart or expiry requires establishment
again; this specification does not define durable ticket-based session resume.

Keepalive is the post-establishment control liveness operation. Its counter must
increase strictly within the active session, independently of data counters.
The requested phase must have both directional keys, or be the permitted next
schedule-2 phase. ACK echoes session, phase and counter and includes server wall
time in milliseconds. Match the outstanding probe; a stale/mismatched ACK does
not establish liveness. Ordinary keepalive is endpoint/state-bound, not AEAD.

SessionClose is best effort, has no success reply, and removes runtime session,
key/replay and route state through the server teardown path. Repeating a close
for already removed state does not create a session. Normal post-init source
binding applies; knowledge of a session id alone is not intended as permission
to close another endpoint's session. There is no reliable close acknowledgement.

## Endpoint binding and NAT rebinding

SessionReady establishes the UDP source endpoint when the source is eligible;
tunnel-inner hairpin sources are not adopted. Post-init policy/keepalive/close
controls from another bound source reject unless the keepalive rebind procedure
below succeeds. Authenticated, valid and successfully delivered DataPacket
traffic may update a return endpoint; malformed, rejected or failed-egress
traffic does not authorize that update.

A client without a usable ordinary endpoint can send a keepalive from its new
source with extension flags 6, a valid auth phase and a proof over an empty
challenge. The server verifies the installed key and emits a challenge-bearing
KeepaliveAck. This does not refresh the session, commit the probe, roll the phase
or replace the endpoint. Repeat the keepalive with flags 7 and the challenge-bound
proof. Only a valid outstanding challenge plus successful normal keepalive
processing commits the endpoint; the resulting ordinary ACK confirms acceptance.

Challenges are 32 random bytes, bound to session, candidate socket address and
current binding generation, valid for 15,000 ms. A binding change invalidates
older challenges. The reference bounds storage to four candidates per session
and 4096 globally, and responses to eight per session/1024 global per 15,000-ms
window. Exact duplicate requests can reuse the outstanding challenge rather than
allocating new state. Invalid proof, stale challenge, changed generation, quota
or full storage results in no success response. Retry/reconnect behavior is local
policy, not a different wire format.

## Key phases and rehandshake

Only schedule 2 supports in-place next-phase rollover. Send a keepalive requesting
the next contiguous U16 phase; use that phase only after the matching ACK.
Skipped phases and exhaustion reject. Both directions derive new keys with the
new phase from the existing shared secret/transcript. Retain the previous phase
for 30,000 ms, then remove its keys and replay window before further ingress.
This rollover does not generate a new DH shared secret. Native scheduling uses
65,536 protected packets or 15 minutes, whichever occurs first.

A fresh-secret replacement uses a new client ephemeral and a strictly larger
client nonce under the same identity/session id. The server keeps the established
session active while the replacement completes policy exchange. SessionReady
selects the next contiguous phase; policy ACK atomically installs the replacement,
with the prior phase retained for the overlap. Downgrading an established
schedule-2 session rejects. A second pending replacement is rejected for the first
5000 ms; a newer valid init can replace stalled pending work after that grace.
This is an implemented exception to older documentation's unconditional overlap
rejection statement.

Opted-in native scheduling initiates fresh-secret replacement at 4,194,304 packets
or 24 hours. Native prepared material has a maximum 60,000-ms lifetime and four
pending entries, single-use activation and oldest-entry eviction. These are
existing implementation policies, not protocol-wide negotiated limits. Failed
replacement leaves the server's previous established state intact until its own
expiry/close, while the client may fail closed and disconnect. Independent
implementations need not reproduce UI counters but must preserve key/counter
identity and ACK-gated cutover.
