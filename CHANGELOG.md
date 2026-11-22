# Changelog — Tina Global Code

Tina Global Code v0.2 Candidate. Status: CANDIDATE / EXPERIMENTAL — NOT AN ADOPTED GLOBAL STANDARD, NOT A GOVERNMENT IDENTITY SYSTEM, NOT A SURVEILLANCE SYSTEM.

DATE PROVENANCE UNRESOLVED (machine timestamps say 2026-11-11; the user asserts 2026-11-06 — unresolved; invent no date).

The v0.1 `generated_at` value (`2026-11-06T10:00:00Z`) is retained in `schemas/schema-manifest.json` because no reliable generation date for v0.2 can be asserted while the date provenance is unresolved; it is the fixture epoch, not a claim about when v0.2 was produced.

## v0.2.0 — Independent-review hardening

This release is the response to the twenty-question independent review (`docs/independent-review.md`). The v0.1 ZIP is untouched; all work is in the source tree. Full test suite: **196 passed, 0 failed**. Conformance: **34/34** (CT-GID-001–034).

### Breaking change (one, deliberate, recorded)

- **Shared context documents carrying `root_ref` now FAIL validation.** The Q14 privacy-attack test proved `root_ref` (SHA-256 of the root ID, byte-identical across contexts, a required field in v0.1) was a keyless, equality-testable join key: the v0.1 claim "unlinkable without the context key" was true of the `derived_id` field and false of the document as a whole. `root_ref` is removed from shared documents entirely. Old `hum_` IDs still resolve; old private records still resolve; only shared documents carrying the join key are rejected. This is the one exception to the century-versioning rule, recorded in `docs/century-versioning.md` and `docs/human-id-opacity-v02.md`. Issuer-side audit linkage now requires the issuer's private-store mapping.

### Added

- **Q1 — class-free shared human ID.** `id_` + exactly 32 lowercase hex chars (first 32 hex of the same HMAC-SHA256 derivation as the legacy `hum_` derived ID, so migration is a pure prefix swap). New schema `opaque-human-id` (0.2.0); `tina_global_code/ids.py`, `tina_global_code/privacy.py`; `docs/human-id-opacity-v02.md`. Old `hum_` IDs remain valid and resolvable; resolvers accept both forms.
- **Q2 — registration authority.** Three-tier allocation logic (`tina_global_code/authority.py`): exactly one GRA; exclusive shard allocation (first local-id character, base-62, 62 shards per namespace); conflict detection and GRA-only resolution; succession/escrow/mirrors on authority disappearance; offline issuance via shard reservations + `reconcile_offline()`; human-identity issuance gate enforced in `Registry.issue()` (GRA or flagged Issuing Authority only — never a device manufacturer, never as a side effect of device registration). New schema `registration-authority` (0.2.0); `docs/registration-authority.md`. CLI: `issue --issuer-kind`.
- **Q3 — real cryptographic signatures.** Ed25519 via the `cryptography` library (required `cryptography>=42`; observed 50.0.2). Profile `ed25519-sha256-v1`; signed bytes = canonical-compact JSON minus the signature field; replaceable profile registry. Key rotation, revocation, expiry, tamper, unknown-issuer, and offline verification (tested with networking disabled). Without the library, verification returns `NOT_VERIFIABLE`, never a fake pass. New schema `crypto-profile` (0.2.0); `tina_global_code/crypto/`; `docs/crypto-profile.md`. The shipped device passport and all three trace events were re-signed with real test-only Ed25519 keys (public key in `fixtures/test-fixture-keys.json`; private key discarded at generation, exists nowhere in the repo).
- **Q4 — device/robot attestation.** Pluggable provider model (`tina_global_code/attestation/`); `SimulatedHardwareProvider` labelled SIMULATED everywhere (it proves the protocol, never real hardware attestation). Fresh-nonce challenge-response + issuer-signed binding record (device id ↔ one key fingerprint ↔ one hardware serial). Clone handling: a clone with a copied key fails on serial mismatch; a clone that copied both key and serial cannot be distinguished by software alone (stated honestly). New schema `attestation` (0.2.0); `docs/attestation.md`.
- **Q5 — tamper-evident registry.** Append-only `registry_events` table with a SHA-256 hash chain and triggers rejecting UPDATE/DELETE; `verify_registry()` replays the event log against the live tables; signed snapshots (HMAC-SHA256); verified backup export/restore. New schema `registry-event` (0.2.0); `docs/tamper-evident-registry.md`. CLI: `verify-registry`, `export-backup`, `restore-backup`, `snapshot`, `verify-snapshot`. Honest limit: a full-log rewrite is exposed only by a signed snapshot/backup held outside the file.
- **Q6 — human credential recovery.** `RecoveryStore` state machine (REQUESTED → AWAITING_HUMAN_VERIFICATION → VERIFIED → OLD_CREDENTIALS_REVOKED → NEW_CREDENTIAL_ISSUED → COMPLETED, plus REJECTED/ESCALATED); all six scenarios tested separately; death/estate ends as `identity_suspended_estate_recorded` (no new credential); code-vs-human-procedure separation documented. New schema `recovery` (0.2.0); `docs/human-credential-recovery.md`.
- **Q9 — foreign robot scenario.** Complete reference scenario (`fixtures/foreign-robot-building/`, 16 files + README; `docs/foreign-robot-scenario.md`) with automated tests, including offline behaviour.
- **Q10 — unknown future namespace.** Synthetic 2076 namespace `qrl_` (explicitly synthetic); the 2026 software parses the envelope, records schema-not-found, grants nothing (default DENY), preserves the entity verbatim, and forwards it unchanged (`docs/unknown-namespace.md`).
- **Q11 — century-scale versioning.** Three-generation simulation (0.1.0 → 0.2.0 → 0.3.0); versioned trace-hash profiles `trace-v0.1` (default separators, as shipped) and `trace-v0.2` (compact canonical); retired namespace `thr_` stays historically resolvable; non-reuse proven non-vacuously (`docs/century-versioning.md`).
- **Q12 — federation failure modes.** Nine conditions classified and tested (`tina_global_code/resolver/federation.py`; `docs/federation-failure.md`): 7× FAILS_CLOSED, 1× WORKS (disconnected building), 2× DEGRADES_SAFELY (partitioned network, regional internet loss), **0 UNSAFE**.
- **Q13 — human/device separation.** Five named errors enforced by the Validator on every document: `HUMAN_CONTROL_CAPABILITY_REJECTED`, `HUMAN_UTILITY_CONTROL_REJECTED`, `HUMAN_ROBOT_COMMAND_REJECTED`, `HUMAN_DEVICE_STATE_REJECTED`, `HUMAN_PERMISSION_INFERRED_REJECTED` — each with a tested legitimate twin that still passes (`docs/human-device-separation.md`).
- **Q15 — AI accountability.** End-to-end test proving all twelve accountability answers are retrievable from trace + passport + evidence; a negative test proves removing VERIFIED makes SUCCESSFUL refused (`docs/ai-accountability.md`).
- **Q16 — capacity stress.** Measured: 10^6 real SQLite rows in 10.9 s (91,411 rows/s); DB 431,407,104 B; lookup mean 0.030 ms, p95 0.057 ms, worst 0.693 ms; batch revocation of 5,000 in 0.04 s; concurrent minting of 4,000 IDs, 0 collisions. 10^9 simulated analytically; 10^11 ≈ 43 TB, 10^15 ≈ 431 PB in one store — hence sharding (`docs/capacity-stress.md`).
- **Q17 — registry authority governance.** Candidate model (`governance/registry-authority-governance-candidate.md`): seven separated bodies; Tina capped at one seat in every body; no veto anywhere for anyone; human-identity rules changeable only by double lock (8/11 supermajority + independent privacy review + 60-day public comment); `hum_` issuable only under a joint charter of ≥3 independent issuing authorities in different jurisdictions; 25% single-vendor funding threshold; 24-month sunset if the Steward is never seated. Compared against GS1, ISO/IEC 15459-2, IANA, DOI/Handle, and W3C DID registries.
- **Q18 — blind independent implementation.** Kept as evidence (not part of the reference package) at `~/workspace/tina-global-code-review/blind-implementation/`, with its own README and INTEROP-REPORT.md.
- **Q19 — spec/implementation consistency audit.** Consolidated SPEC GAP and IMPLEMENTATION GAP lists in `docs/independent-review.md` §Q19 (30 SPEC GAP items, 12 fixed/closed in v0.2, 16 open, 1 known wart, 1 packaging note; 13 IMPLEMENTATION GAP items).
- **Namespaces.** `mdl`, `skl`, `tol` restored from the Tina v1.1 baseline (they were missing from the current namespace table, so migrated IDs were rejected by the validator). **42 namespaces** (was 39). Schema discovery updated for all three.
- **CLI.** New commands: `verify-registry`, `export-backup`, `restore-backup`, `snapshot`, `verify-snapshot`; `--issuer-kind` on `issue`.

### Fixed (defects found during the independent review, with regression tests)

1. **Execution-schema inference mis-route.** The validator checked authorization/action-request before execution, so an execution document was mis-routed to the authorization schema and rejected. Fixed by checking execution first (`test_execution_schema_inference_fixed`).
2. **`mdl`/`skl`/`tol` namespaces missing** (see above; `test_model_namespace_registered_v02`).
3. **Context derivation was MD5("hello").** The shipped v0.1 derived ID was literally `MD5("hello") = 5d41402a…`. Regenerated with real HMAC-SHA256; pharmacy derived ID is now `hum_2728af9442ff3f69da7671728f68f9b6` / `id_2728af9442ff3f69da7671728f68f9b6` (key `b"reference-only-test-key"`, input UTF-8(root)||0x00||UTF-8(context), first 32 hex). Test: `test_shipped_example_is_the_real_derivation`.
4. **`root_ref` join key** (see Breaking change above).
5. **Trace-hash contradiction.** README (default separators) vs provenance doc (compact canonical) gave different hashes for the same document. Fixed by versioned profiles; the README formula reproduces all three shipped trace hashes exactly under `trace-v0.1`.
6. **Shipped passports failed their own extension schemas** (12 ext schemas required `entity_id`/`entity_type` the examples don't carry). Fixed; "exactly one" dispatch now enforced (`maxProperties: 1`, error code `PASSPORT_EXTENSION_DISPATCH_REJECTED`).
7. **Passport signatures unverifiable** ("SHA-256" listed as a signature algorithm). Fixed by real Ed25519 signatures; "SHA-256" remains in the schema enums as a deliberate backward-compatibility wart (old values verify as `NOT_VERIFIABLE`).
8. **Events asserted `actual` at REQUESTED/AUTHORIZED.** `actual` removed from the two shipped requested/authorized event examples and forbidden by a validator rule before EXECUTED (`test_event_actual_forbidden_before_execution`).
9. **Resolver-response `query_id` pattern forbade truthful answers about unknown namespaces.** Now an unconstrained string echoing the query verbatim.
10. **Capability reason text** said "22 capabilities"; the schema enum lists 23. Fixed.
11. **Passport doc** said "Core fields (all required unless noted)"; the schema requires nine. Fixed, and the `signature.algorithm` line corrected (Ed25519/ECDSA-P256 only; SHA-256 is a content hash, not a signature algorithm).
12. **Capacity doc** pointed at a nonexistent "issuance policy" for the shard mechanism. Now points at `docs/registration-authority.md` §2.
13. **Validator elimination path** — `schema_name` is now set to the elimination winner so semantic checks always run.
14. **`verify-snapshot` CLI crashed** (`AttributeError: 'Namespace' object has no attribute 'snapshot_id'`) because its positional was declared as `"snapshot-id"`. Fixed by declaring it `snapshot_id`; regression test `test_cli_verify_snapshot_and_verify_registry` drives snapshot → verify-snapshot → verify-registry through the real CLI.
15. **Registry `issue()` double wall-clock** and **`verify_registry()` `KeyError` on malformed event payload** (now reported as tampered). Fixed in the registry-hardening work.

### Changed

- Schema manifest: 52 entries (46 + 6 new: attestation, crypto-profile, opaque-human-id, recovery, registration-authority, registry-event); 24 at 0.2.0, 28 at 0.1.0. Bumped to 0.2.0: passport-core, trace-event, context-identity, namespace, global-id, resolver-response, and all 12 `*-passport-ext` schemas.
- Package version: `0.2.0` (`tina_global_code/__init__.py`, CLI docstring, README title/status). Dependencies: Python standard library + `jsonschema` (+ `cryptography` for signature verification, optional — without it verification returns `NOT_VERIFIABLE`, never a fake pass). No network use.
- `docs/independent-review.md` answers all twenty review questions (verdict + evidence + limits per question). **No final status is picked in this release** — the evidence relevant to each of the four candidate statuses is laid out in Q20 for Tina to decide.

### Known gaps (still open — see `tests/remaining-failures.md` for the full list)

- No CLI path to mint a `hum_` ID (no `--issuer-record` flag; API only).
- Signature verification in the Validator is opt-in (a KeyStore must be supplied); without one, signatures stay presence-only with a warning.
- Authentication is explicitly out of scope. No policy-evaluation engine, no federation wire protocol, no key registry/key discovery, no registry discovery mechanism, no evidence `content_hash` input definition, no normative freshness/staleness defaults, no valid examples for ten document types, no document→schema dispatch discriminator.
- Registry-side human/device separation enforcement (validator-only today).
- Attestation exists only as a SIMULATED provider; no real hardware attestation.
- No governance body is seated; no Registration Authority is appointed.

## v0.1.0 — Candidate (baseline)

38 unit tests, 34/34 conformance, 16/16 valid examples, 12/12 invalid rejected with a reference time. ZIP `tina-global-code-v0.1-candidate.zip` (278,424 B, SHA-256 `d925de30938ad04cd6f1e97afdc89a3932eb395185746d8865caf5e413f5c8ec`). Final status picked by Tina: IMPLEMENTATION CANDIDATE — TESTS PASS, READY FOR INDEPENDENT REVIEW. The v0.1 record is preserved in `tests/TEST-RESULTS.md` ("v0.1 record (kept)").
