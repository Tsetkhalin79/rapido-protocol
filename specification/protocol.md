# Scope, versions and provenance

## Reading this specification

This is a factual interoperability specification of the pinned implementation,
not a security endorsement. MUST, MUST NOT and REQUIRED describe bytes or behavior
needed for the documented compatible path. Explicit observations about permissive
decoders do not require new implementations to reproduce every permissive edge.
Unsupported or unverified behavior is named rather than inferred.

The scope includes frame codecs, session establishment, device proof, directional
key derivation, data protection, rekey, endpoint rebinding and UDP/WSS transport.
Account creation, billing, credential enrollment APIs, catalog hosting, routing
installation and server deployment are provisioning concerns. An independent
peer must arrange a valid enrolled identity and any required trusted server key
before using an account-backed server. The wire protocol does not grant accounts
or negotiate tunnel interface addresses.

## Version and capability registry

The only accepted header version is `1`. The only control codec is `bin_v1`:
frame codec bits equal `2`; negotiation byte `2` means BinV1 and `0` means absent.
Legacy JSON codec bits `0` are rejected as frame codec selection. There is no
general extension TLV or unknown-tail skipping: optional tails have the exact
order described in [packet format](packet-format.md).

| Registry | Values |
| --- | --- |
| Handshake profile | 0 bootstrap; 1 transcript-bound HKDF; 2 static/ephemeral X25519 |
| Traffic key schedule | 0 bootstrap HKDF; 1 transcript-bound HKDF; 2 X25519 plus transcript HKDF |
| Transform preference | 0 baseline; 1 supported selection metadata |

Profile and schedule are distinct fields, but the server selects matching pairs
0/0, 1/1 or 2/2. Unknown ids in either offered list reject the handshake.
An empty list supports the legacy 0 fallback, not every algorithm. Profile 1 does
not specify an additional data cipher or a new frame layout: its selection is
bound into schedules 1/2. The inspected data paths use the common frame/AEAD layout.
Do not implement an imagined transform wrapper from its name.

Repository defaults disable both upgraded server handshake gates and client
schedule-2 opt-in; clients normally advertise `[0,1]`. Thus code support for
schedule 2 is not evidence that a shipped build or deployed server selects it.

## Evidence inventory

All references below identify paths at source commit
`084ddc73ddd9b442ef56c5af0ddd80430140536e`. They are provenance, not prerequisites
for reading or implementing this specification; no private repository link is
needed. The snapshot identity excludes later source changes. Local routing and
pre-transport packet selection are outside the wire protocol described here.

| Subject | Source paths / functions |
| --- | --- |
| Existing baseline | `docs/PROTOCOL_RFC_V0.md`; `docs/PRODUCTION_HANDSHAKE_MIGRATION.md` |
| Types and constants | `contracts/src/lib.rs`; `defaults/protocol.json`; `defaults/server.json`; `defaults/client.json` |
| Frame and binary control codec | `protocol/src/lib.rs`: `FrameHeader`, `decode_frame`, `encode_binary_payload`, `decode_binary_payload`, `decode_data_plane_packet` |
| Key schedules and data protection | `server/src/lib.rs`: `derive_bootstrap_data_plane_traffic_key`, `derive_production_v1_directional_data_plane_traffic_key`, `forward_secret_transcript_hash`, `derive_forward_secret_handshake_material`, `derive_forward_secret_v1_directional_data_plane_traffic_key`, `data_plane_nonce`, `data_plane_aad` |
| Credential verification and endpoint binding | `server/src/main.rs`: `session_init_device_credential_transcript`, `verify_session_init_device_credential_proof`, `evaluate_source_rebind_keepalive`, UDP ingress loop |
| Session transitions and replay | `server/src/session_handshake.rs`: `select_handshake`, `SessionManager`, `maybe_roll_forward_secret_key_phase` |
| WSS carrier | `server/src/wss_gateway.rs`; `client/src/features/session/sessionBootstrap.ts`; corresponding Windows bootstrap |
| Android data-plane derivation | `client/android/app/src/main/java/com/rapidovpn/app/RapidoVpnService.kt`: `deriveBootstrapDataPlaneKey`, `deriveProductionV1DataPlaneKey`, `deriveForwardSecretV1DataPlaneKey` |
| iOS data-plane derivation | `client/ios/RapidoPacketTunnel/PacketTunnelProvider.swift`: same three derivation families; `client/ios/RapidoClientTemplate/AppDelegate.swift`: control exchange |
| Windows data-plane derivation | `client-windows/windows/RapidoClientWindows/WindowsTunnelAgentControl.h`: `DeriveForwardSecretV1DataPlaneKey` and bootstrap/production derivations |
| Identity trust provisioning | `shared/client/forwardSecretHandshake.cjs`; `shared/client/catalogAuthenticity.cjs`; `defaults/forward_secret_server_identity_policy.json` |
| Existing vectors | `shared/protocol/session_interop_fixtures.json`, `session_interop_negative_fixtures.json`, `forward_secret_interop_fixtures.json`; protocol unit/property tests; native interop tests |

This repository provides no SDK or executable examples. Historical roadmap
assertions remain historical; only the behavior explicitly described here
belongs to this snapshot.
