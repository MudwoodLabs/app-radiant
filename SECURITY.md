# Security Policy

## Scope

This document describes the threat model and known limitations of the community-built Radiant Ledger Nano S Plus app (`Zyrtnin-org/app-radiant`). Applies to v0.0.6 beta and later until superseded.

## Reporting a Vulnerability

If you discover a security issue that could expose user funds, allow signature forgery, or bypass the path-lock, **do NOT open a public GitHub issue**. Email the maintainer directly or reach out on the Radiant Discord with a DM request for a secure channel.

Less-severe issues (UX confusion, documentation errors, non-exploitable bugs) can be filed as public GitHub issues.

## Threat Model

### Assets protected

1. **User's private keys** — derived from their BIP39 seed inside the Ledger secure element. Must never leave the device.
2. **User's signing intent** — the transaction the user approves on-screen must be what the device actually signs ("what you see is what you sign").
3. **SLIP-44 coin namespace** — keys derived under `m/44'/512'/...` must not be reachable by other coin apps (BTC, BCH, etc.) that may share the seed.

### Actors considered

- **Malicious host software** (compromised wallet, malicious script, compromised USB bus) — can send arbitrary APDUs to the device.
- **Malicious transaction inputs** — host claims an input has value X, includes scriptCode Y, sequence Z, etc. Device only sees what the host sends.
- **Passive network observer** — can see on-chain txs once broadcast; cannot influence signing.

### Out of scope

- **Physical attack on the device** — hardware tamper, side-channel, JTAG, etc. The BOLOS secure element handles this.
- **Supply-chain compromise of the build environment** — we rely on Docker digest-pinning for reproducibility. If the pinned image is compromised, so are all downstream builds.
- **Compromised Ledger firmware itself** — the app runs inside the Ledger's secure element and relies on firmware guarantees.
- **User giving away their 24-word recovery phrase** — no app can protect against this.

## Security Guarantees (What v0.0.6 Provides)

### 1. Path-lock on all key-touching handlers

Keys derived under this app are restricted to `m/44'/512'/...` (Radiant's SLIP-44 coin type). Three independent enforcement points:

- [`handler/hash_sign.c`](lib-app-bitcoin/handler/hash_sign.c) — transaction signing
- [`handler/get_wallet_public_key.c`](lib-app-bitcoin/handler/get_wallet_public_key.c) — pubkey export / address derivation
- [`handler/sign_message.c`](lib-app-bitcoin/handler/sign_message.c) — message signing (added in v0.0.6 — defense-in-depth)

A malicious host that asks for a signature under `m/44'/0'/...` (BTC) or `m/44'/145'/...` (BCH) receives `SW_INCORRECT_DATA` (0x6A80) from the app.

### 2. Radiant sighash correctness

The device's `hashOutputHashes` computation matches the radiantjs reference implementation and Radiant mainnet consensus. Triple-validated:

- **Python oracle** ([`radiant-ledger-app/scripts/radiant_preimage_oracle.py`](https://github.com/Zyrtnin-org/radiant-ledger-app/blob/main/scripts/radiant_preimage_oracle.py)) — port of radiantjs `sighash.js` + `script.js`.
- **18 mainnet fixtures verified** — published ECDSA signatures from 5 real mainnet txs verify against oracle-computed sighashes.
- **Device-vs-oracle hardware tests** — plain P2PKH and Glyph-output signing both produce signatures that verify against the oracle.

This means a malicious host cannot trick the device into signing a different tx than what the user approved on-screen.

### 3. Output script classification

Outputs not matching one of these shapes are rejected with `SW_TECHNICAL_PROBLEM_2` (0x6F0F):

- Standard P2PKH (25 bytes exactly): `76 a9 14 <hash160> 88 ac`
- Glyph-P2PKH (any length ≥ 25 bytes with the P2PKH pattern at offsets 1-3 and 24-25): `<glyph-prefix> 76 a9 14 <hash160> 88 ac <glyph-suffix>`
- OP_RETURN with zero amount (specific cases only, not exposed in v1.0)
- P2SH (rejected by the Radiant coin variant intentionally — v2 scope)

The Radiant classifier guards against OOB-read on short scripts by requiring `script_len >= 0x19` before pattern-matching (added v0.0.6).

### 4. State isolation between signing sessions

`radiant_output_hash_reset()` unconditionally reinitializes all per-tx state (hash contexts, ref accumulator, opcode walker state, satoshi accumulator) on:

- Transaction cancel / user decline
- Error path (SW returned, tx discarded)
- Start of a new transaction

Verified by explicit test: state from tx A does not leak into tx B's sighash.

### 5. Bounds on state consumption

Hard limits designed to prevent resource exhaustion or adversarial state inflation:

- `MAX_OUTPUT_TO_CHECK = 100` bytes per output (effective cap on Glyph script size — see Known Limitations)
- `RADIANT_MAX_PUSH_REFS = 8` unique refs per output (9+ rejected with `SW_INCORRECT_DATA`)
- 8-byte script-length varint (0xFF prefix) rejected — practically, Radiant scripts can't exceed 4GB anyway
- `MAX_BIP32_PATH = 10` levels (inherited from upstream app-bitcoin)

## Known Limitations

### 1. Community-built; "Not genuine" banner

Ledger's secure-bootloader displays a persistent **"This app is not genuine"** banner any time this app is open. This is expected for any community-developed Ledger app and cannot be removed without Ledger's review process. It does NOT indicate compromise — verify by comparing `bin/app.sha256` to the release SHA256.

### 2. No independent security audit

The review that led to v0.0.6 was conducted by the core developer with AI-assisted analysis. **No third-party security professional has audited this code.** For any application where you'd want formal audit assurance (institutional custody, large-balance treasury), this is not the right tool.

### 3. Single-firmware validation

The app has been tested on one Nano S Plus firmware version (the current stable as of 2026-04-16). Behavior across firmware versions is unverified. If you run an older or newer firmware, please report any differences.

### 4. MAX_OUTPUT_TO_CHECK = 100 caps real script size

Although `RADIANT_MAX_SCRIPT_PUBKEY = 10000` in `helpers.c`, the upstream `hash_input_finalize_full.c` buffer (`currentOutput`) is 100 bytes. Outputs producing more than ~75 bytes of script fail with `SW_INCORRECT_DATA` before reaching the Radiant FSM. In practice this accommodates:

- All plain P2PKH outputs (25 bytes)
- Single-ref Glyph-P2PKH (63 bytes)
- OP_RETURN memos up to ~80 bytes (not exposed in v1.0)

Multi-ref Glyph outputs exceeding ~75 bytes cannot be spent in v1.0. Tracked as a future enhancement.

### 5. ~~Conflicting push-ref opcodes not detected~~ — fixed in v0.0.7

Previously: if a script had `OP_PUSHINPUTREF <ref>` and `OP_DISALLOWPUSHINPUTREF <same_ref>` in the same output, the Python oracle rejected it (matching radiantjs consensus) but the device accepted it silently. As of v0.0.7, the device tracks disallow-refs in a parallel accumulator and rejects with `SW_INCORRECT_DATA` at output-emission time if any overlap with push-refs is detected.

### 6. ~~Integer wrap in PUSHDATA4 length~~ — fixed in v0.0.7

Previously: a script with `OP_PUSHDATA4` declaring a 4GB payload could wrap the skip counter. Contained by outer bounds so non-exploitable, but state-machine hygiene was poor. As of v0.0.7, all length-taking opcodes (direct push 0x01-0x4B, OP_PUSHDATA1/2/4, push-ref opcodes 0xD0/D1/D2/D3/D8) validate that the declared payload fits in the remaining script bytes before entering skip/read state.

### 7. Wallet-side UX gaps

`Electron-Wallet@radiant-ledger-512` (the patched host wallet) does not yet recognize Glyph-prefixed P2PKH UTXOs. Users spending Glyph UTXOs must use the CLI harness documented in `radiant-ledger-guide`. Tracked as two issues on the `Electron-Wallet` repo.

## Verification / Reproducibility

### Cross-check the published binary

```bash
cd app-radiant
git checkout v0.0.7
git submodule update --init --recursive

docker run --rm -v "$(pwd):/app" -u "$(id -u):$(id -g)" \
  ghcr.io/ledgerhq/ledger-app-builder/ledger-app-builder-lite@sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834 \
  bash -c "cd /app && make COIN=radiant BOLOS_SDK=\$NANOSP_SDK"

sha256sum bin/app.hex
# Expected: 7e51dbff88a42752fdff333ee0f26ace9e740e7381c99737ead1f65da7318f0f
```

If your SHA256 doesn't match the release, either:
- You're on a different tag — `git log --oneline -1` should match the release's advertised commit
- Your Docker image is not the pinned digest — verify `docker images --digests`
- Reproducibility is broken — report as a security issue

### Cross-check the Python oracle

```bash
cd radiant-ledger-app/scripts
python3 oracle_self_validate.py        # 3-way validation — exit 0 = PASS
python3 test_oracle_against_vectors.py # 5 mainnet fixtures — exit 0 = PASS
python3 test_push_refs.py              # 22 unit tests — exit 0 = PASS
```

No hardware required. If any of these fail on a clean clone, the oracle has a bug.

### Cross-check device signing

```bash
# Requires hardware: Nano S Plus, Radiant app sideloaded, patched Electron-Wallet
cd radiant-ledger-app/scripts
python3 test_device_plain_sign.py   # Plain P2PKH — sig verifies against oracle
python3 test_device_glyph_sign.py   # Glyph output — sig verifies against oracle
```

## References

- [`radiant-ledger-guide` troubleshooting section](https://github.com/Zyrtnin-org/radiant-ledger-guide#10-troubleshooting) — SW code table
- [`radiant-ledger-app/docs/solutions/`](https://github.com/Zyrtnin-org/radiant-ledger-app/tree/main/docs/solutions) — compound fix docs including all security findings from v0.0.5 → v0.0.6 hardening
- [radiant-node consensus source](https://github.com/RadiantBlockchain/radiant-node) — canonical reference for Radiant sighash + opcode semantics

## Version History

- **v0.0.7** (2026-04-16): Disallow-ref vs push-ref conflict detection, PUSHDATA/push-ref bounds checks (closes remaining FSM audit findings)
- **v0.0.6** (2026-04-16): `sign_message` path-lock, `output_script_is_regular` OOB guard, Radiant-branded icons
- **v0.0.5** (2026-04-16): Glyph opcode walker, `output_script_is_regular` relaxation for Glyph
- **v0.0.3** (2026-04-15): `hashOutputHashes` preimage field (first mainnet-accepted Ledger-signed Radiant tx)
