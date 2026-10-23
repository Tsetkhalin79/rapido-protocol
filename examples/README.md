# Data-only interoperability material

These JSON documents are data, not SDKs or a reference implementation. Field
names describe decoded messages; JSON is not the accepted UDP control codec.

- [Session messages](session_interop_fixtures.json): copied existing synthetic
  message objects for a legacy/tail-free session; no real user credential.
- [Negative session cases](session_interop_negative_fixtures.json): copied
  existing objects and local expected errors; not wire error responses.
- [Schedule-2 KDF vector](forward_secret_interop_fixtures.json): copied existing
  deterministic test inputs and expected outputs. Server private bytes are
  repeated `0x07`; client private bytes are repeated `0x09`. All private/shared/
  traffic values are **public synthetic test material**, never production keys.
  Its proof string is a placeholder and will not pass managed-account admission.

All three originate from `shared/protocol/` at the source commit identified in
[provenance](../specification/protocol.md). Their byte content is preserved.
They must never be used as deployed credentials or trust roots.

## Minimal annotated frame

Synthetic, newly authored Ping with nonce 1, no padding:

```text
01 01 00 08 00 02 00 00 00 00 00 00 00 00 00 01
```

The first 8 bytes are version 1, kind 1, payload length 8, BinV1 flags 2,
reserved 0. The final 8 bytes are the unsigned big-endian nonce. A corresponding
Pong changes only kind byte `01` to `02`. This example is a codec illustration,
not an authenticated session or data-encryption vector.
