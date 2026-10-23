# RAPIDO-PROTOCOL-EXTRACTION

## Current state

Objective: extract the existing Rapido wire specification into its sole canonical
repository without runtime changes or remote publication.

The initial **PASS covered the bounded file extraction and local checks**, not
complete factual lifecycle agreement. Independent review of the frozen initial
candidate `8ddc18fd0352fba7c06ca10361cfc134653bd53d` returned PARTIAL with
two P2 documentation findings, R-001 and C-001. This correction round updates
the affected lifecycle text; it has not received a new independent verdict.
Neither result is a production-security or end-to-end interoperability
certificate. Publication belongs to the Owner.

## Scope and result inventory

Source snapshot: `084ddc73ddd9b442ef56c5af0ddd80430140536e` on the original
`feature/windows-safety-loss-watch` branch. Work is isolated on
`feature/protocol-spec-extraction-20261023`. The original dirty checkout and its
index are preserved. Inspected uncommitted work concerns IPv4 bypass/local routing,
native service ownership and application documentation; it is not included in the
source snapshot. No claim is made about deployed binaries or their configuration.

Target branch: `intellisys/protocol-spec-extraction` in the prepared repository.
The target had no initial commit and no project-local documentation/rules.

- Extracted and corrected material from `docs/PROTOCOL_RFC_V0.md` and
  `docs/PRODUCTION_HANDSHAKE_MIGRATION.md`, verified against server and native
  implementation paths listed in [provenance](specification/protocol.md).
- Added README, six focused specification pages, licensing notice, official
  license, this report and an examples guide with one synthetic annotated frame.
- Copied three existing synthetic JSON fixtures without changing their bytes.
- Source changes are limited to four Markdown files: the RFC pointer, migration
  maintenance page, documentation index descriptions and project-status notice.
  Product target requirements, rollout evidence, test/release/provisioning
  obligations and existing text-guard markers are retained as non-wire material.

## Findings and completeness

The [discrepancy table and limits](specification/errors.md) are part of the result.
Material findings: wire version 1 versus historical v0 naming; binary control
versus stale JSON wording; kind 11 and optional tails absent from old summaries;
whole-frame size limit; idempotent controls; manual WSS versus hypothetical
automatic QUIC/TCP; stalled-rehandshake replacement; security properties below.

The subsequent independent PARTIAL identified two additional lifecycle gaps:

- **R-001 (P2):** The pinned server has one expiry entry for the immediately
  previous phase. A second rollover inside the 30,000-ms overlap overwrites that
  entry without deleting the older key or replay state. Valid old-phase data can
  pass the retained-key/replay boundary after that period; forwarding still
  depends on later checks. The revised [phase lifecycle](specification/handshake.md#key-phases-and-rehandshake)
  and [limits](specification/errors.md#remaining-limits-and-concrete-unknowns)
  disclose that the overlap is not a global retirement bound. This is a static
  source trace, not a running-server reproduction or runtime remediation.
- **C-001 (P2):** A source-eligible, decodable keepalive requesting the next
  schedule-2 phase installs it before the server checks the replay counter. A
  duplicate/lower counter then rejects without ACK or TTL refresh, but the new
  active phase can be used on subsequent server return data. The revised
  [keepalive lifecycle](specification/handshake.md#state-transitions-and-expiry)
  and [rejection table](specification/errors.md#error-handling) distinguish this
  server side effect from the client's ACK-gated send requirement. This does not
  assert transactional rollback or specify behavior for every malformed frame.

Schedules 0/1 derive traffic keys solely from public handshake fields. Schedule 2
uses a custom client-ephemeral/server-static X25519 construction and lacks forward
secrecy against later compromise of the server static private key. These findings
follow directly from `server/src/lib.rs` derivation functions at the pinned commit
and matching Android/iOS/Windows KDF paths. They are conditional on selecting those
schedules; no deployed-state claim is made. Defaults keep upgraded server gates
and client schedule-2 opt-in disabled. No runtime remediation was performed.

Independent codec/key-schedule implementation is supported by explicit encodings,
ordered transcripts, labels, nonce/AAD, state transitions and carrier framing.
Public-service account provisioning/catalog API integration, exhaustive native
malformed-input parity and a complete valid managed-account capture remain outside
the established evidence. Existing fixtures are not a full interoperability
certificate. See the concrete limits in the linked specification.

## Publication and licensing

Owner approved CC BY 4.0 for specification prose and data-only examples on
2026-10-23. Attribution is Intellisys Software LLC. LICENSE is the official full
legal code downloaded from `https://creativecommons.org/licenses/by/4.0/legalcode.txt`.
Application code/dependencies are not relicensed. The repository inspection is
not proof of contributor-assignment or patent clearance.

Only protocol prose and synthetic data are transferred. Synthetic fixture private
keys/shared secrets/traffic keys are explicitly public test inputs (repeated 0x07
and 0x09 private bytes), not operational secrets. No production configuration,
real infrastructure endpoint, personal account data, internal organization record,
private filesystem location or third-party implementation is included in target
content. Required protocol labels, paths and numeric constants are retained.

## Initial extraction verification and local delivery

One targeted documentation verification cycle was completed:

- All 14 target files inspected; local Markdown links resolve, and there are no
  local anchor links requiring resolution. No Cyrillic authored Markdown found.
- All three copied JSON fixtures parse and are byte-identical to their source.
- The schedule-2 transcript SHA-256 and both directional HKDF results were
  independently recomputed from the documented field order and fixture inputs;
  all match the existing vector. X25519 itself was not recomputed: the local
  optional cryptography package was unavailable. No dependency was installed.
- Manual publication review plus pattern checks found no operational credentials,
  live infrastructure addresses, internal filesystem locations or organization
  records. Synthetic fixture key material is intentionally retained and labeled.
- Source diff is exactly four Markdown files. All 13 existing shell-guard literal
  expectations for the two extracted documents remain satisfied. No local source
  documentation links targeting removed anchors were found. Executable sources,
  package files, build inputs, scripts and fixtures in Rapido-VPN are unchanged.
- Whitespace defects and one wrapped guard phrase found during the cycle were
  corrected; only affected checks were repeated. An initial checker file-count
  assumption was corrected from 15 to the actual 14 without changing content.
- The original checkout remains on its original branch/HEAD with 58 preexisting
  dirty paths, untouched by this task. The isolated worktree contains only the
  four scoped documentation changes before commit.

The source pre-commit hook runs source-formatting, Clippy and broad Rust tests.
It is bypassed only for this documentation commit with command-local Git
configuration to honor the directive's no-runtime-change/no-full-suite boundary;
no persistent hook setting is changed. Target has no configured project hook.
No full product build/tests, independent review, new automation, push, PR, release
or remote mutation was run during the initial extraction.
Final commit identities are delivered with the handoff rather than embedded
self-referentially in this committed report. The Owner decides publication.

## Documentation correction verification

The correction task compares only the two revised lifecycle descriptions with
the pinned server paths: `session_handshake.rs` phase installation/pruning and
keepalive counter ordering, `lib.rs` phase-key/data replay lookup, and `main.rs`
active-phase selection for return data. The directly affected Markdown was
checked for contrary unconditional expiry or rejection-rollback claims, local
links/anchors, English text and whitespace. The correction changes only
`specification/handshake.md`, `specification/errors.md`,
`specification/security.md` and this report in the target repository. The
security page clarifies its counter-wrap wording and links to phase advancement.
No fixture bytes, crypto formulas or source files change, so the earlier fixture
and cryptographic checks are reused rather than repeated.
These are author-side targeted checks, not independent acceptance. No push, PR,
merge, release or publication is part of this correction.
