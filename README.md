# Tina Global Code v0.2 Candidate — reference implementation

Tina Global Code v0.2 Candidate. Status: CANDIDATE / EXPERIMENTAL — NOT AN ADOPTED GLOBAL STANDARD, NOT A GOVERNMENT IDENTITY SYSTEM, NOT A SURVEILLANCE SYSTEM.

DATE PROVENANCE UNRESOLVED (machine timestamps say 2026-11-11; the user asserts 2026-11-06 — unresolved; invent no date).

Machine-readable world identity standard (candidate), reference
implementation, experimental. Core chain: IDENTITY → SEMANTIC MEANING →
CAPABILITY → AUTHORITY → POLICY → ACTION → PHYSICAL OR DIGITAL EFFECT →
VERIFICATION → EVENT → EVIDENCE → AUDIT.

Rules: a human is not a device; identity does not imply authority;
authority does not imply execution; execution does not imply success;
success does not exist without verification. Completion boundary
(strict): REQUESTED ≠ AUTHORIZED ≠ EXECUTED ≠ PHYSICALLY EFFECTIVE ≠
VERIFIED ≠ SUCCESSFUL.

This repository does **not** claim global adoption, government approval,
perfect privacy or security, legal compliance everywhere, 100 billion
IDs assigned, humans controlled, that every object requires a Tina ID,
or that Tina replaces existing standards. Capacity figures (≥ 10^11
per namespace) are syntactic capacities, not population or inventory
claims.

## Layout

```
tina_global_code/          Python package (stdlib + jsonschema +
                           cryptography for v0.2 signature verification)
  __init__.py              status line, version
  constants.py             namespaces, dispatch table, tau families, enums
  canonical.py             canonical JSON, SHA-256, trace event_hash formula,
                           replaceable crypto profiles
  crypto/                  Ed25519 signature verification (profiles,
                           keys, signing, key store, verdicts)
  attestation/             device/robot attestation (SIMULATED provider
                           only in v0.2; pluggable provider registry)
  ids.py                   Global ID parsing/validation, capacity arithmetic,
                           minting, legacy v1.1 reference parsing
  privacy.py               context-specific derived human IDs (HMAC-SHA256,
                           REFERENCE-ONLY — caller-supplied keys)
  authority.py             static permission/delegation checks, grid-signal
                           rule, completion-boundary check, registration
                           authority (allocation, human-issuance gate)
  migration.py             Tina v1.1 -> Global Code migration adapter
  validator/               schema + semantic validation, trace verification,
                           evidence-level checks; LIMITATIONS.md;
                           crypto_checks.py (standalone signature/
                           attestation checks, opt-in wired)
  registry/                sample local registry on embedded SQLite;
                           tamper-evident event log, recovery store
  resolver/                local -> cache -> federated (stub) resolver;
                           federation.py (failure-mode decisions)
  cli/                     `tina` command-line interface
tests/                     unittest suite + conformance suite (CT-GID-001..)
security/                  threat model, crypto profiles, security test cases
governance/                candidate governance + change control
schemas/ contexts/ examples/ fixtures/ docs/ migrations/
                           (read-only inputs — not edited by this implementation)
tina                       entry script (execs `python3 -m tina_global_code.cli`)
registry/                  sample registry database lives here when created
                           (registry/tina-global-code.db; created on first
                           use, never pre-filled)
```

The top-level `validator/`, `resolver/`, `cli/` directories are
reserved placeholders; the implementation lives in the
`tina_global_code` package. The top-level `registry/` directory holds
the sample database file only.

## Requirements

- Python 3.10+ (developed on 3.12.3)
- `jsonschema` 4.10.3
- `cryptography` >=42 (required for v0.2 signature verification and
  attestation; without it, verification returns NOT_VERIFIABLE)
- No internet at runtime; no network calls are made anywhere.

## Quick start

```sh
# Validate a document (reference time = fixture epoch, for the shipped
# examples dated 2026-11-06; without it, staleness is checked against
# wall-clock now — see "Reference time" below)
./tina --reference-time 2026-11-06T10:05:00Z validate examples/valid/valid-passport-device.json
./tina validate-passport examples/valid/valid-passport-human.json
./tina verify-trace examples/valid/valid-trace-chain-3.json
./tina verify-evidence examples/valid/valid-passport-device.json

# Registry: seed, issue, resolve, inspect (uses a temp DB with --db)
./tina --db /tmp/demo.db seed
./tina --db /tmp/demo.db issue --namespace dev --authority org_7e21aa09c4d5e6f78
./tina --db /tmp/demo.db resolve bld_9f2c7a41e8b64d2a91c3
./tina --db /tmp/demo.db inspect bld_9f2c7a41e8b64d2a91c3
./tina --db /tmp/demo.db register examples/valid/valid-revocation.json
./tina --db /tmp/demo.db revoke dev_a1b2c3d4e5f60718 --reason "smoke test" --by org_7e21aa09c4d5e6f78

# Migration from Tina v1.1
./tina migrate-v1 <v1.1-document.json>

# Schemas
./tina schema passport-core

# Tests and conformance
./tina test            # full unittest suite, PASS/FAIL per test
./tina conformance     # CT-GID suite, writes tests/conformance-report.json
python3 tests/run_tests.py
python3 tests/conformance.py
```

Every command exits 0 on success and non-zero on failure. `--json`
gives machine-readable JSON output. `--db PATH` selects the registry
database (default `registry/tina-global-code.db`).

## Reference time

Staleness (`is_stale`, `max_age_seconds`) and delegation/permission
expiry are only meaningful against a reference time. Pass
`--reference-time <ISO-8601>`; without it, expiry is checked against
wall-clock now and staleness checks are skipped with a warning. The
shipped examples are dated 2026-11-06, so validating them against
wall-clock time days later correctly flags their measurements as
stale — use `--reference-time 2026-11-06T10:05:00Z` to validate the
fixtures at their own epoch.

## Trace event_hash formula (v0.1)

```
event_hash = SHA-256( json.dumps({
    "sequence", "run_id", "actor_id", "timestamp",
    "previous_hash", "event_kind", "public_payload"
}, sort_keys=True).encode("utf-8") )
```

Default JSON separators (not the compact canonical form) — this
reproduces the shipped valid trace fixture exactly. The general
canonical form (sorted keys, separators `(",", ":")`, no whitespace)
is used for sealed commitments and content hashes. Unifying the two
is a v0.2 change-control item; see `security/crypto-profiles.md`.

## Known discrepancies found while building (not silently fixed)

1. **Namespace count.** The v0.1 docs and brief said "38 namespaces";
   the actual v0.1 enumeration was **39**. v0.2 registers three more
   (`mdl_`, `skl_`, `tol_`, restored from the Tina v1.1 baseline), so
   the count is now **42**. The enumeration is authoritative.
2. **Tau family count.** The migration doc says "23 listed"; the
   family enum it prints contains **25**. The enum is authoritative.
3. **Capability count.** One invalid example's reason text says "22
   capabilities"; the schema enum lists **23**. The schema is
   authoritative.
4. **Supersedes conflict.** The migration doc says to record a
   superseded `ppt_<object-id>` in `supersedes`, but passport-core
   requires supersedes items to match the Global ID pattern, and a
   nested `ppt_<global-id>` does not match. The adapter records the
   old ID in `provenance.method` and in the migration report instead;
   see `tina_global_code/migration.py` and
   `governance/change-control.md` item 4.
5. **Trace hash serialization.** See the formula above; the fixture
   uses default separators while the general canonical form is
   compact. Documented, not silently changed.

## What this implementation cannot prove

See `tina_global_code/validator/LIMITATIONS.md` — physical
effectiveness, real-world authority, live remote revocation, key
compromise, signature validity, staleness without a reference time,
and more. A passing validation is a statement about documents, not
about the world.
