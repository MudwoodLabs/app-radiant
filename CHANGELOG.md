# Change Log

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

---

## Radiant fork (community) — `appVersion` ≠ upstream `APPVERSION_*` numbers

## [0.0.8-glyph-transfer] — 2026-04-26 (pre-release)

**Scope**: Additional security fixes from the 2026-04-20 AI-assisted audit, plus closure of the Glyph NFT transfer-preserving spend path. All fixes are in `lib-app-bitcoin` (`radiant-v1` branch, `a7f6195`..`d169caa`, tagged `v0.0.8-glyph-transfer`).

### Security

- **M3** (`a7f6195`): `output_script_is_regular` and `output_script_p2pkh_offset` both matched `0xD0` (Glyph FT shape) alongside `0xD8` (NFT singleton). FT outputs don't carry a pkh at the same offset as NFT outputs; any on-device review or change-address match on an FT output used the wrong offset. Fix: restrict both helpers to `0xD8` only. FT support deferred to v0.0.9.

- **M1** (`ea60dc2`): `output_script_p2pkh_offset` decoded `buffer[1..4]` without first checking `buffer[0] >= 0x19`. A script shorter than 25 bytes could reach the offset path. Fix: return 0 if `buffer[0] < 0x19`.

- **M6 / L2** (`32528c8`): `output_script_is_op_return` read `buffer[1]` and `buffer[2]` without verifying the script length. A 0-byte or 1-byte script caused an out-of-bounds read. Fix: return 0 if `buffer[0] == 0`. Closes the M6 audit finding listed in v0.0.5.

- **L1** (`d169caa`): `hash_input_finalize_full.c` had a change-detection fallback that set `addressOffset = OUTPUT_SCRIPT_REGULAR_PRE_LENGTH` (4) when `output_script_p2pkh_offset` returned 0, then still ran the 20-byte `memcmp`. An unrecognized output shape could accidentally match the change address at the wrong offset. Fix: gate the `memcmp` on `addressOffset != 0`; unrecognized shapes are now correctly skipped as non-change candidates.

### Changed

- **L3** (`7e40367`): Corrected misleading comments in `output_script_is_regular` that said scripts of "any length" were accepted; the `buffer[0] >= 0x19` minimum-length check is now documented accurately.

### Known unfixed audit findings (pre-release caveats)

- **H1** — short-script classifier spoofing via stale `currentOutput` bytes in `output_script_is_regular` / `output_script_p2pkh_offset` (requires plumbing `scriptSize` to the helpers).
- **M5** — `discardTransaction` echoes up to 200 bytes of attacker-supplied `currentOutput` back to host in the error APDU reply.
- **M9** — `bip44_derivation_guard` operator-precedence bug in the ternary expression (upstream-inherited).

### Mainnet proofs after this release

- [`af0cd27d…6201c9`](https://explorer.radiantblockchain.org/tx/af0cd27d6201c9) — first Ledger-signed Glyph NFT transfer-preserving spend (singleton ref carried forward to output)

---

## [0.0.5-security-fixes] — 2026-04-20 (pre-release)

**Scope**: security-audit remediation. Audit was AI-assisted (five parallel reviewers); report: [`SECURITY_AUDIT_2026-04-20.md`](https://github.com/Zyrtnin-org/Electron-Wallet/blob/glyph-ft-all/SECURITY_AUDIT_2026-04-20.md). No second human reviewer on the diffs yet — pre-release status pending community code review.

### Security

- **B3** (`lib-app-bitcoin` 0967950): `check_output_displayable` now calls `output_script_p2pkh_offset` for the change-address memcmp offset. For 63-byte Glyph-wrapped P2PKH outputs (`d8|d0 <ref36> 75 76a914 <pkh20> 88ac`), the pkh is at offset 42; the old hardcoded `OUTPUT_SCRIPT_REGULAR_PRE_LENGTH=4` landed inside the push-ref region. An attacker who knew the victim's change-pkh could embed those 20 bytes inside a crafted ref, causing the firmware to mark a funded output as change (`displayable=false`) — silently hidden from on-device review while still counted in `totalOutputAmount`. Fund-diversion class.

### Known unfixed audit findings (pre-release caveats)

- **H1** — short-script classifier spoofing via stale `currentOutput` bytes in `output_script_is_regular` / `output_script_p2pkh_offset` (requires plumbing `scriptSize` to the helpers).
- **M5** — `discardTransaction` echoes up to 200 bytes of attacker-supplied `currentOutput` back to host in the error APDU reply.
- **M6** — `output_script_is_op_return` reads `buffer[1]` / `buffer[2]` without length check (0-byte script OOB).
- **M9** — `bip44_derivation_guard` operator-precedence bug in the ternary expression (upstream-inherited).

### Wallet side (`Zyrtnin-org/Electron-Wallet@glyph-ft-all`)

- **B1** — wallet-side `Transaction.verify_signature` against the locally-recomputed sighash for every input before applying the device signature. Pre-broadcast detection of any wallet↔firmware sighash divergence.
- **B2** — fix per-output refsHash sort to raw byte-lex (matches firmware `memcmp` and Python oracle). Prior reversed-byte sort happened to agree on single-ref outputs; first multi-ref output would have failed consensus.
- **B4** — hard-fail in `add_input_info` when the parent tx of a Glyph input isn't in the wallet store. Previously fell back silently to `p2pkh` type → tx signed locally but rejected at mempool.
- **B5** — 8-test `TestGlyphNftCommands` suite mirroring the FT command tests.

---

## [0.0.4-glyph-ft-transfer] — 2026-04-20

### Added

- `MAX_OUTPUT_TO_CHECK` raised from 100 → 200 in `lib-app-bitcoin/context.h`. Enables 3-output Glyph FT transfers (recipient + FT change + RXD change).
- Diagnostic SW codes `0x6FB1..0x6FB5` per `handle_output_state` reject branch.

### Mainnet proofs after this release

- [`5d5b2600d0…f047390`](https://explorer.radiantblockchain.org/tx/5d5b2600d0f06c35f67778f8f103a8b8ff86bef49d99d7172afc6db12f047390) — first Ledger-signed Glyph FT transfer (3-output)
- [`a323dfc543…15a3be1`](https://explorer.radiantblockchain.org/tx/a323dfc543834eaf035a273b2d0b9f545683085c8ec7202af0e25a16715a3be1) — first wallet-integrated Ledger-signed NFT transfer

---

## [0.0.3-sighash-fix] — 2026-04-15

### Fixed

- Radiant `hashOutputHashes` sighash preimage insertion between `hashSequence` and `hashOutputs`. First post-fix mainnet Ledger-signed Radiant tx: [`de3574979f…56893743`](https://explorer.radiantblockchain.org/tx/de3574979f986616b4152c4294b85562318292490d3587d8fe32aff456893743).

---

## Upstream LedgerHQ/app-bitcoin history (pre-fork)

## [2.4.10] - 2026-02-19

### Modified

- Derivation Path Hardening:
    - `HAVE_APPLICATION_FLAG_DERIVE_MASTER` is removed for all coins except Bitcoin Legacy and Bitcoin Test Legacy
    - BIP-32 derivation paths are enforced using wildcard syntax (`m/*/<COIN_TYPE>`), with a few exceptions for Bitcoin forks
- Ticker moved on the right.

## [2.4.9] - 2025-09-08

### Modified

- Apex P porting
- Nano S support removal

## [2.4.8] - 2025-08-08

### Fixed

- Adaptation to IO revamp changes.

## [2.1.0] - 2022-09-15

### Modified

Rename the "bitcoin" target to "bitcoin_legacy".

## [1.6.6] - 2022-05-11

Technical release for the deployment of the legacy app.

## [1.6.2](https://github.com/ledgerhq/app-bitcoin/compare/1.6.1...1.6.2) - 2021-06-24

### Fixed

- Fixed Qtum derivation for Native Segwit accounts
- Revert Firo's COINID to zcoin
## [1.6.1](https://github.com/ledgerhq/app-bitcoin/compare/1.6.0...1.6.1) - 2021-05-31

### Modified

- Zcoin becomes Firo
## [1.6.0](https://github.com/ledgerhq/app-bitcoin/compare/1.5.6...1.6.0) - 2021-04-30

### Added

- Better python tests
- Add wallet ID feature on Nano X
## [1.5.6](https://github.com/ledgerhq/app-bitcoin/compare/1.5.5...1.5.6) - 2021-03-25

### Added

- Compatibility with Nano S 2.0.0 firmware
- Message signing displays the whole message hash instead of truncating it

## [1.5.5](https://github.com/ledgerhq/app-bitcoin/compare/1.5.4...1.5.5) - 2021-01-06

### Added

- Support for Native Segwit on VertCoin

## [1.5.4](https://github.com/ledgerhq/app-bitcoin/compare/1.5.3...1.5.4) - 2021-01-06

### Fixed

- Remove a change that was breaking swap feature when used with older apps [#180](https://github.com/LedgerHQ/app-bitcoin/pull/180)

### Added

- Tests and GitHub Actions CI

## [1.5.3](https://github.com/ledgerhq/app-bitcoin/compare/1.5.2...1.5.3) - 2020-12-11

### Fixed

- Fix pin validation check on Nano X

## [1.5.2](https://github.com/ledgerhq/app-bitcoin/compare/1.5.1...1.5.2) - 2020-12-10

### Added

- Changelog file

### Removed

- unused `prepare_full_output` and `btchip_bagl_confirm_full_output` functions removed

### Changed

- More errors, less THROWs
- Cleanup args parsing when called as a library

### Fixed

- Most compilation warnings fixed
- Ensure `os_lib_end` is called when errors are encountered in library mode
- Fix pin validation check
