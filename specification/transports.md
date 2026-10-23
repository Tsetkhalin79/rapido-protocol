# UDP and HTTPS/WSS carriers

## UDP

One datagram contains one complete Rapido frame. There is no stream length
prefix outside the common header, batching envelope, reliable-delivery layer or
application fragmentation in this snapshot. Use the configured server UDP port;
the port is endpoint configuration, not a cryptographic constant. Reordering and
loss are possible. Retransmission of policy controls follows the lifecycle's
idempotency rules; retransmission of an already accepted encrypted data counter
is a replay and is dropped.

The source supports optional random auth padding. There is no implemented QUIC
carrier implied by old roadmap text. Ping/Pong exists as a codec/diagnostic pair;
it does not establish an authenticated session or replace SessionKeepalive.

## HTTPS mode

The client's HTTPS mode is **WebSocket over TLS**, not an HTTP POST carrying IP
packets. Use a provisioned `wss://` endpoint, normally path `/rapido/vpn/v1`, and
offer subprotocol `rapido-vpn-v1`. The reference gateway requires the exact
configured path, rejects a query string, and returns that subprotocol in the
upgrade response. A wrong path is HTTP 404; a missing required subprotocol is
HTTP 400. Standard TLS hostname/certificate validation and standard WebSocket
framing/masking apply. A preexisting HTTP health/config endpoint is not an
alternative data carrier.

Each binary WebSocket **message** is exactly one unchanged Rapido frame, in
either direction. There is no additional Rapido stream prefix. Do not concatenate
frames or send JSON/text messages. WebSocket fragmentation is handled by the
WebSocket implementation; it is not Rapido handshake fragmentation.

The reference Rust gateway is a local plaintext WebSocket service behind TLS
termination, creates a separate UDP relay socket per WebSocket connection and
forwards binary messages to the same session/data runtime. Do not expose that
plaintext listener as an encrypted endpoint. The relay socket is the runtime's
source endpoint; reconnecting a WebSocket can therefore require a fresh session
or authenticated endpoint recovery. Carrier connectivity alone is not proof that
the Rapido policy ACK or data keys have been accepted.

Reference configured limits: 2048 bytes per WebSocket message/frame, 131072-byte
write buffer, 256 connections, 2000 client messages per second with burst 4000,
and a 5000-ms HTTP upgrade timeout. They do not enlarge the 1200-byte Rapido
encoder budget. Deployment may configure these limits. Empty binary/text and
rate-policy violations close with 1008; oversize binary messages use 1009 (the
WebSocket library can reject an oversize frame before application handling).
Close ends the carrier. WebSocket Ping/Pong is carrier control, counted in the
gateway message budget, and does not refresh Rapido session state.

## Selection and recovery

The inspected client bootstrap defaults to `connectionMode = udp`; explicit
`https` requires a selected endpoint with a WSS URL, otherwise it reports
`https_endpoint_unavailable`. It passes the WSS URL/subprotocol to the native
transport. The wire format has no "switch to HTTPS" control message and no
universal negotiated UDP-to-WSS failover threshold. Manual mode choice and client
reconnect policy must not be described as an implemented automatic QUIC/TCP
fallback protocol. Establishment, identity checks and data protection remain
required in either carrier.

## IP packet and routing boundary

The server validates a nonempty IPv4/IPv6 packet, its declared length and policy
MTU after AEAD. IPv4 needs at least 20 bytes, IHL at least 20 and within the packet,
and exact total length. IPv6 needs at least 40 bytes and an exact payload length
match. Invalid packets are dropped. Destination routing, OS TUN setup, forwarding
and NAT are outside the wire exchange. No-egress is a valid fail-closed server
configuration; successful handshake then does not imply Internet connectivity.

The implementation can allocate node-local authenticated-device IPv4 leases and
translate legacy client addresses before TUN delivery, restoring addresses on
return. This is server-local isolation; no lease address is negotiated in
SessionReady or PolicySnapshot. Platform route exclusions and local bypass
selection likewise do not modify the packet cipher/codec.

Return packets use the server-to-client key and independent counter. A bounded
server return queue may replay an already protected frame after authenticated
endpoint migration; it retains its original phase/counter and normal replay
rules. It is not a reliable-delivery or duplicate-acceptance extension.
