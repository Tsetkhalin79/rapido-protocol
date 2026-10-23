# Rapido Protocol

Rapido carries client IP packets in session-bound binary frames over UDP or a
WebSocket over TLS carrier (called HTTPS mode by the clients). This repository
is the canonical specification source. It describes the existing implementation;
it does not introduce a new protocol or provide an SDK.

**Specification:** extraction revision 2026-10-23, draft implementation snapshot.
**Wire version:** 1. Historical documents called this protocol baseline "v0";
that name is not the first header byte. The source snapshot is Rapido-VPN commit
`084ddc73ddd9b442ef56c5af0ddd80430140536e`. No deployment or current released-client
compatibility is certified by this extraction.

The server admits an identity, negotiates a key schedule, exchanges the tunnel
policy, and installs directional data keys after the policy acknowledgement.
Protected data frames then carry individual IPv4/IPv6 packets. WSS preserves the
same inner frames and session state machine.

Read the [security limitations](specification/security.md) before implementing:
schedules 0 and 1 derive keys from public transcript fields, and the implementation
name `noise_nk_x25519_chachapoly_sha256` does not establish conformance to standard
Noise NK or forward secrecy against server-static-key compromise.

- [Scope, versioning and provenance](specification/protocol.md)
- [Binary frame and message formats](specification/packet-format.md)
- [Handshake, lifecycle and endpoint rebinding](specification/handshake.md)
- [Authentication, exact key schedules and security limits](specification/security.md)
- [UDP and HTTPS/WSS transports](specification/transports.md)
- [Errors, discrepancies and remaining gaps](specification/errors.md)
- [Data-only examples and test vectors](examples/README.md)
- [Extraction result and publication checks](EXTRACTION_REPORT.md)

Copyright 2026 Intellisys Software LLC. The specification text and data-only
examples are licensed under [CC BY 4.0](LICENSE), approved by the Owner on
2026-10-23. See [licensing scope](LICENSING.md). This does not license the separate
application implementation. Publication remains a separate Owner action.
