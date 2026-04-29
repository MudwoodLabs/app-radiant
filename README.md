# Radiant Ledger App (Community)

**Sideloadable Ledger Nano S Plus app for signing Radiant (RXD) transactions.**

Hardware-wallet signing for the Radiant blockchain. Private keys stay on-device. Forked from [LedgerHQ/app-bitcoin](https://github.com/LedgerHQ/app-bitcoin); adds a `radiant` Makefile variant and an on-device `hashOutputHashes` computation that Radiant consensus requires (and BCH's Ledger app does not produce).

---

## Status: BETA

Mainnet proofs — all three asset classes signed by this app:

| Asset | First Ledger-signed mainnet tx | Date |
| --- | --- | --- |
| Plain RXD | [`de3574979f…56893743`](https://explorer.radiantblockchain.org/tx/de3574979f986616b4152c4294b85562318292490d3587d8fe32aff456893743) | 2026-04-15 |
| Glyph NFT singleton | [`af0cd27d9c…6201c9`](https://explorer.radiantblockchain.org/tx/af0cd27d9cda2113cc9882274ff7015f09f759ffe8b71b0c17e86c64fb6201c9) | 2026-04-16 |
| Glyph FT transfer (3-output, with FT change) | [`5d5b2600d0…f047390`](https://explorer.radiantblockchain.org/tx/5d5b2600d0f06c35f67778f8f103a8b8ff86bef49d99d7172afc6db12f047390) | 2026-04-20 |

Current tag: **`v0.0.5-security-fixes`**

Community-distributed. Not reviewed by Ledger. The device will show a persistent **"This app is not genuine"** banner when the app is open — that is expected for any unsigned community-built Ledger app and cannot be removed.

---

## What works (v1 scope)

- **Nano S Plus** — other devices not supported yet
- **P2PKH sends and receives** (standard `1…` Radiant addresses)
- **SLIP-44 coin type 512**, BIP44 derivation path `m/44'/512'/0'/0/x`
- Integration with a [patched Electron Radiant](https://github.com/MudwoodLabs/Electron-Wallet/tree/radiant-ledger-512) (branch `radiant-ledger-512`)
- Reproducible CI builds; SHA256s published on every release

## What works (v0.0.5 additions)

- **Security audit remediation** — see Electron-Wallet's `SECURITY_AUDIT_2026-04-20.md`. Fixes `check_output_displayable` to use the Glyph-wrapper-aware `output_script_p2pkh_offset` helper instead of a hardcoded offset, closing a class of change-address matching bug where attacker-crafted ref bytes could trick the firmware into hiding a funded output from the user's on-device review.
- **Unique diagnostic SW codes** (0x6FB1..0x6FB5) per `handle_output_state` reject branch, so hosts can distinguish firmware-reject reasons without a PRINTF-enabled build.

## What works (v0.0.4 additions)

- **Glyph NFT transfers** (63-byte singleton template)
- **Glyph FT transfers** — full-balance sends (2 outputs) and partial sends with FT change (3+ outputs), via `MAX_OUTPUT_TO_CHECK=200` buffer

## What doesn't work yet (v2 scope)

- P2SH destinations (`3…` addresses) and OP_RETURN memos
- Glyph dMint / mint-authority spends (≥241-byte scripts exceed `MAX_OUTPUT_TO_CHECK`)
- Nano X, Stax, Flex

## Important: derivation path differs from Samara/Electron/Chainbow

Existing Radiant wallets use `m/44'/0'/...` (Bitcoin's SLIP-44 coin type). This Ledger app uses **`m/44'/512'/...`** per the SLIP-0044 registry entry for RXD. If you want to move RXD onto this Ledger, send from your existing wallet's address to your new Ledger-derived address. Standard "upgrading to hardware wallet" flow — the old wallet isn't locked out, you just move the coins.

---

## Install

Prerequisites on Linux:

```bash
pip install ledgerblue
wget -q -O - https://raw.githubusercontent.com/LedgerHQ/udev-rules/master/add_udev_rules.sh | sudo bash
# Unplug + replug device
```

Build the app (requires Docker, ~2GB builder image download on first run):

```bash
git clone --recurse-submodules https://github.com/MudwoodLabs/app-radiant.git
cd app-radiant
git checkout v0.0.5-security-fixes
git submodule update --init --recursive

docker run --rm \
  -v "$(pwd):/app" \
  -u "$(id -u):$(id -g)" \
  ghcr.io/ledgerhq/ledger-app-builder/ledger-app-builder-lite@sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834 \
  bash -c "cd /app && make COIN=radiant BOLOS_SDK=\$NANOSP_SDK"
```

Sideload to device (Nano S Plus unlocked, on dashboard):

```bash
python3 -m ledgerblue.loadApp \
  --targetId 0x33100004 \
  --targetVersion="" \
  --apiLevel 25 \
  --tlv \
  --curve secp256k1 \
  --path "44'/512'" \
  --appFlags 0x0 \
  --fileName bin/app.hex \
  --appName "Radiant" \
  --appVersion "0.0.5" \
  --dataSize 512 \
  --installparamsSize 64 \
  --delete
```

Approve prompts on-device ("Allow unsafe manager" then "Install app Radiant from unverified source"). See [`BUILDER.md`](BUILDER.md) for reproducibility details.

## Use with Electron Radiant

Patched plugin lives at [`MudwoodLabs/Electron-Wallet@radiant-ledger-512`](https://github.com/MudwoodLabs/Electron-Wallet/tree/radiant-ledger-512). Clone, run from source, wizard defaults will pick up `m/44'/512'/0'` automatically.

**Important workflow order**: open the Radiant app on the device BEFORE you open the wallet in Electron Radiant. If the device is on the dashboard when Electron Radiant tries to talk to it, you'll get an `SW 6702` error.

---

## Looking for testers

Have a Nano S Plus and some spare RXD? Open an issue here or ping on the Radiant Discord to get the pre-release install working on your device. We're validating across firmware versions and tx shapes before wider release.

## Related repos

- [`MudwoodLabs/lib-app-bitcoin`](https://github.com/MudwoodLabs/lib-app-bitcoin) branch `radiant-v1` — submodule with the `hashOutputHashes` C implementation
- [`MudwoodLabs/Electron-Wallet`](https://github.com/MudwoodLabs/Electron-Wallet) branch `radiant-ledger-512` — host-side wallet plugin
- [`MudwoodLabs/radiant-ledger-app`](https://github.com/MudwoodLabs/radiant-ledger-app) — planning, Python oracle, golden-vector fixtures, investigation notes

---

## Technical background

Radiant's signature preimage inserts a 32-byte `hashOutputHashes` field between `nSequence` and `hashOutputs` ([`radiant-node/src/script/interpreter.cpp:2636-2650`](https://github.com/RadiantBlockchain/radiant-node/blob/master/src/script/interpreter.cpp#L2636)). BCH's signing path doesn't produce this field, so stock BCH-family Ledger apps produce signatures that Radiant mainnet rejects.

This app's C diff extends `lib-app-bitcoin` to compute `hashOutputHashes` on-device from the streaming output bytes it already hashes for the standard `hashedOutputs` field. Zero additional host-trust introduced. Full arc documented at [`radiant-ledger-app`](https://github.com/MudwoodLabs/radiant-ledger-app).

---

## Credits

- Base app forked from [LedgerHQ/app-bitcoin](https://github.com/LedgerHQ/app-bitcoin)
- Radiant network: [RadiantBlockchain](https://github.com/RadiantBlockchain)
- Radiant JS preimage reference: [radiantjs](https://github.com/RadiantBlockchain/radiantjs)

License: Apache-2.0 (inherited from upstream app-bitcoin).

---

<details>
<summary>Upstream LedgerHQ/app-bitcoin README (preserved for reference)</summary>

# Ledger Legacy Bitcoin Application

## Legacy bitcoin application

Bitcoin wallet application for Ledger devices up to version 1.6.5.

> **Warning**
> This is currently only used in order to support and maintain altcoins cloned from Bitcoin.
> The last stable version of the app as it was used for Bitcoin is kept in the branch [legacy-1.6.6](https://github.com/LedgerHQ/app-bitcoin/tree/legacy-1.6.6) for future reference and does not support devices starting Stax.
>
> Versions starting from 2.0.0 are at https://github.com/LedgerHQ/app-bitcoin-new.

Ledger Blue is not maintained anymore, but the app can still be compiled for this target using the branch `blue-final-release`.

The original beta specification can be found at https://ledgerhq.github.io/btchip-doc/bitcoin-technical-beta.html — with the regular set of APDUs for standard wallet operations enabled.

</details>
