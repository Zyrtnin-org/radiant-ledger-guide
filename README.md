# Radiant Ledger Hardware Wallet Guide

**Complete Guide to Using a Ledger Nano S Plus with the Radiant Blockchain (RXD)**

This guide documents how to install, use, and develop for the community-built Radiant Ledger app. Covers sending RXD, receiving and spending Glyph NFT UTXOs, wallet integration for developers, and the Radiant-specific sighash preimage format for researchers.

Designed to be used as context for AI coding agents (Claude, Cursor, etc.) — paste the README into your session and start working. See [BUILDING_WITH_CLAUDE.md](BUILDING_WITH_CLAUDE.md) for AI-assisted workflow tips.

> **FOR AI AGENTS — Start Here:**
> - **Installing the app on your Ledger?** → Read sections 2 (Install & Sideload) and 3 (Device Setup)
> - **Sending RXD from a Ledger?** → Read section 4 (Your First Send)
> - **Received a Glyph NFT and want to spend it?** → Read section 6 (The Glyph Spend Gotcha) — this is important, the GUI can't do it
> - **Building a wallet that integrates the Ledger?** → Read section 7 (For Wallet Developers)
> - **Verifying the build or porting to another chain?** → Read section 8 (For Researchers) and section 9 (Reproducibility)
> - **Hit an error code?** → Jump to section 10 (Troubleshooting)

---

## Table of Contents

1. [Status & Scope](#1-status--scope)
2. [Install & Sideload](#2-install--sideload)
3. [Device Setup: Pair with Electron Radiant](#3-device-setup-pair-with-electron-radiant)
4. [Your First Send (RXD, plain P2PKH)](#4-your-first-send-rxd-plain-p2pkh)
5. [Receiving Glyph NFTs to a Ledger Address](#5-receiving-glyph-nfts-to-a-ledger-address)
6. [The Glyph Spend Gotcha](#6-the-glyph-spend-gotcha)
7. [For Wallet Developers: Integrating Ledger Support](#7-for-wallet-developers-integrating-ledger-support)
8. [For Researchers: Radiant's Sighash Preimage](#8-for-researchers-radiants-sighash-preimage)
9. [Reproducibility: Build & Verify Yourself](#9-reproducibility-build--verify-yourself)
10. [Troubleshooting](#10-troubleshooting)
11. [Appendix: Quick Reference](#11-appendix-quick-reference)

---

## 1. Status & Scope

### What Works (v1.0)

- **Nano S Plus only** (other devices are v2 scope)
- **P2PKH sends + receives** — standard `1...` Radiant addresses
- **Glyph NFT receiving** — when someone else mints a Glyph (via software signing) and sends it to your Ledger address, the UTXO arrives correctly on-chain
- **Glyph NFT spending** — your Ledger can sign a tx that consumes a Glyph-bearing UTXO already held by a Ledger address (via direct-APDU harness; see [section 6](#6-the-glyph-spend-gotcha))
- **SLIP-44 coin type 512**, BIP44 derivation path `m/44'/512'/0'/0/x`
- **Reproducible CI builds**, SHA256s published per release

### What's v2 Scope (Not in v1.0)

- Glyph NFT **minting** from a Ledger (requires non-standard scriptSig the app doesn't yet support)
- P2SH destinations (`3...` addresses)
- OP_RETURN memo outputs
- SIGHASH_SINGLE + SIGHASH_ANYONECANPAY
- Schnorr signatures
- Nano X, Stax, Flex device support
- Official Ledger listing

### Current Release

- **Tag**: `v0.0.7` (beta)
- **First mainnet-confirmed P2PKH tx**: [`de3574979f986616b4152c4294b85562318292490d3587d8fe32aff456893743`](https://explorer.radiantblockchain.org/tx/de3574979f986616b4152c4294b85562318292490d3587d8fe32aff456893743)
- **First mainnet-confirmed Glyph spend**: [`22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3`](https://explorer.radiantblockchain.org/tx/22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3)

### Repository Layout

| Repo | Purpose |
|------|---------|
| [`Zyrtnin-org/app-radiant`](https://github.com/Zyrtnin-org/app-radiant) | Main Ledger app — fork of LedgerHQ/app-bitcoin |
| [`Zyrtnin-org/lib-app-bitcoin`](https://github.com/Zyrtnin-org/lib-app-bitcoin) | Submodule with the C diff (`hashOutputHashes` computation, path-lock, script rules) |
| [`Zyrtnin-org/Electron-Wallet`](https://github.com/Zyrtnin-org/Electron-Wallet) | Host-side wallet plugin (Electron Cash fork with Radiant-specific patches) |
| [`Zyrtnin-org/radiant-ledger-app`](https://github.com/Zyrtnin-org/radiant-ledger-app) | Planning artifacts, Python oracle, golden-vector fixtures |
| **This repo** | Community usage + developer integration guide |

---

## 2. Install & Sideload

### Prerequisites (Linux)

```bash
# Python tooling for sideload
pip install ledgerblue

# Udev rules so the device is accessible without sudo
wget -q -O - https://raw.githubusercontent.com/LedgerHQ/udev-rules/master/add_udev_rules.sh | sudo bash

# Unplug + replug device after installing udev rules
```

### Option A: Reproducible Build From Source (Recommended)

```bash
git clone --recurse-submodules https://github.com/Zyrtnin-org/app-radiant.git
cd app-radiant
git checkout v0.0.7
git submodule update --init --recursive

docker run --rm \
  -v "$(pwd):/app" \
  -u "$(id -u):$(id -g)" \
  ghcr.io/ledgerhq/ledger-app-builder/ledger-app-builder-lite@sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834 \
  bash -c "cd /app && make COIN=radiant BOLOS_SDK=\$NANOSP_SDK"
```

Expected output: `bin/app.hex` with a reproducible SHA256 hash.

Cross-check the SHA256 against the published value for your tag. If it matches, the binary you built is identical to what others built.

### Sideload to Device

Unlock the Nano S Plus, navigate to the **dashboard** (no app open), then:

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
  --appVersion "0.0.6" \
  --dataSize 512 \
  --installparamsSize 64 \
  --delete
```

On-device prompts (approve each):

1. **"Allow unsafe manager"** — required for community apps
2. **"Uninstall app Radiant"** — only if you're upgrading from an earlier version
3. **"Install app Radiant from unverified source"** — **verify the hash shown on-device matches the `bin/app.sha256` file locally**
4. **"Perform installation"**

After install, open the Radiant app. It will display a persistent **"This app is not genuine"** banner — this is expected for any community-built Ledger app and cannot be removed without Ledger's review.

---

## 3. Device Setup: Pair with Electron Radiant

The stock Electron Radiant wallet doesn't support the Ledger out of the box. Use the patched fork:

```bash
git clone https://github.com/Zyrtnin-org/Electron-Wallet.git
cd Electron-Wallet
git checkout radiant-ledger-512

# Create a venv and install requirements
python3 -m venv .venv
source .venv/bin/activate
pip install -r contrib/requirements/requirements.txt
pip install -r contrib/requirements/requirements-hw.txt
pip install hidapi

./electron-radiant
```

### Wallet Creation

1. Open Electron Radiant → **File → New/Restore**
2. Give it a name (e.g., `ledger-wallet`)
3. Select **Standard wallet** → **Next**
4. Select **Use a hardware device** → **Next**
5. **Open the Radiant app on your Ledger first**, then click "Rescan"
6. Select the Ledger, pick derivation path `m/44'/512'/0'` (should be the default)

> **Important workflow order**: open the Radiant app on the device **BEFORE** you open the wallet in Electron Radiant. If the device is on the dashboard when Electron Radiant tries to talk to it, you'll get an `SW 6702` error.

### Derivation Path Differs from Samara/Chainbow

Existing Radiant wallets use `m/44'/0'/...` (Bitcoin's SLIP-44 coin type, legacy). This Ledger app uses **`m/44'/512'/...`** per the SLIP-0044 registry entry for RXD. If you want to move RXD onto this Ledger, send from your existing wallet's address to your new Ledger-derived address. Standard "upgrading to hardware wallet" flow — the old wallet isn't locked out, you just move the coins.

---

## 4. Your First Send (RXD, plain P2PKH)

1. Fund your Ledger address via a normal send from your existing wallet (start small, like 1 RXD, to validate)
2. Wait for 1 confirmation
3. In Electron Radiant with Ledger connected + Radiant app open:
   - **Send** tab
   - Paste destination `1...` address
   - Enter amount
   - **Preview** — Electron shows the tx details
   - **Send** — device lights up, shows output address + amount for confirmation
4. **Approve on device** (check the address + amount match what you intended)
5. Electron Radiant broadcasts automatically after device returns the signature

### Expected Device Flow

On the Nano S Plus during approval:

```
Output #1
1LkYcHBg...VFhMRc
```
Press right to scroll through the full address, then:

```
Amount
0.05 RXD
```
Press both buttons to **Approve**.

Then:

```
Fees
0.022 RXD    (example — Radiant fee market runs ~10k sats/byte)
```
Press both buttons to **Approve**.

Final screen:

```
Sign transaction?
[REJECT]     [ACCEPT]
```
Press both buttons on **ACCEPT**.

### Fee Math (Radiant-Specific)

Radiant's minimum relay fee is **0.1 RXD/kB = 10,000 sats per byte**. A typical 1-input 2-output plain P2PKH tx is ~225 bytes → minimum fee ~2.25M sats (0.0225 RXD).

This is ~10,000× Bitcoin's typical rate. Don't be alarmed — it's the network norm.

---

## 5. Receiving Glyph NFTs to a Ledger Address

Glyphs cannot be minted from a Ledger (see section 1 — the reveal-tx scriptSig is non-standard). But you CAN have someone else mint on your behalf and send the resulting Glyph UTXO to a Ledger-derived address.

To receive a Glyph NFT on your Ledger:

1. Derive a Ledger address (e.g., `m/44'/512'/0'/0/3`)
2. Give that address to whoever is minting the Glyph (a wallet, marketplace, or a tool like FlipperHub). They do the commit + reveal transactions using software signing (see [`radiant-glyph-nft-guide`](https://github.com/Zyrtnin-org/radiant-glyph-nft-guide) for how minting works)
3. They set the reveal-tx's output destination to your Ledger address
4. Once broadcast, the Glyph UTXO appears at your address on mainnet

### What to Expect

- The Glyph UTXO has a **non-standard scriptPubKey** (typically 60-80 bytes):

```
OP_PUSHINPUTREFSINGLETON (0xd8)   1 byte
<ref>                             36 bytes   (32B commit-txid + 4B commit-vout LE)
OP_DROP (0x75)                    1 byte
<standard P2PKH>                  25 bytes   (76a914 <hash160> 88ac)
```

Total: 63 bytes for the simplest Glyph shape.

- **Electron Radiant will NOT show this UTXO in the Coins tab.** The wallet's script parser doesn't recognize this shape as "belongs to my address." You can still verify receipt on-chain via a block explorer (e.g., [Radiant Explorer](https://explorer.radiantblockchain.org/)).
- The balance shown in Electron Radiant will be missing the Glyph UTXO's value (typically dust, 1-10k sats, so not noticeable unless the Glyph carries significant RXD).

---

## 6. The Glyph Spend Gotcha

**Spending a Glyph-bearing UTXO with a Ledger is possible but NOT supported by the Electron Radiant GUI as of v0.0.7.**

Why: Electron Cash's script classifier only recognizes standard P2PKH, P2SH, and OP_RETURN shapes. A Glyph-prefixed P2PKH (63 bytes) is classified as "unknown script type" and never associated with the owning address. The wallet can't construct a spend because it doesn't see the UTXO.

### How to Spend Today

**Validated path** (what the v1.0 release uses): Run the `spend_real_glyph_2in.py` harness from [`radiant-ledger-app/scripts/`](https://github.com/Zyrtnin-org/radiant-ledger-app/blob/main/scripts/spend_real_glyph_2in.py). It drives the Ledger via direct APDU to spend the Glyph UTXO. Mainnet proof: [`22d4e0e07200…`](https://explorer.radiantblockchain.org/tx/22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3).

This path is CLI-only and requires you to know the UTXO's txid, vout, and the path your Ledger holds it at. Acceptable for power users and for one-off transfers; not a great UX for regular users.

### Future Path (Wallet Integration)

For a normal Send-tab flow, wallets need a Glyph-aware script parser that links Glyph-prefixed P2PKH UTXOs to the owning address. See [section 7](#7-for-wallet-developers-integrating-ledger-support) for the pattern — roughly 20 lines of code. Once a wallet lands this, the GUI will work normally. Not implemented in `Electron-Wallet@radiant-ledger-512` yet; tracking via GitHub issues.

### Key Constraints for Spending a Glyph UTXO

#### Constraint A: The Glyph UTXO Usually Can't Pay Its Own Fee

Radiant's 10k sats/byte min relay means a typical 1-input 1-output spend is ~191 bytes → fee = 1.91M sats. If the Glyph UTXO only holds 0.01 RXD (1M sats), **the UTXO cannot pay its own spend fee.**

**Fix**: add a second input (a plain P2PKH UTXO you control) to cover the fee gap:

```
Input 0: Glyph UTXO     (1.08M sats)
Input 1: Plain P2PKH    (5.00M sats)
Output:  Plain P2PKH    (2.58M sats = 6.08M - 3.50M fee)
Fee:     3.50M sats     (10,355 sats/byte for 338-byte 2-in 1-out tx)
```

#### Constraint B: btchip's `inputIndex` Is Position-Within-List

When signing a multi-input tx, you sign one input at a time:

```python
for i in range(num_inputs):
    app.startUntrustedTransaction(False, 0, [chip_inputs[i]], scriptCode_i, version=0x02)
    #                                     ^-- inputIndex=0 (position in THIS list)
    sig = app.untrustedHashSign(paths[i], lockTime=0, sighashType=0x41)
```

**Always use `inputIndex=0`** when the list has one element, regardless of which tx input you're actually signing. `inputIndex=i` (where `i` is the tx-input position) silently signs with empty scriptCode, producing an invalid preimage.

#### Constraint C: scriptCode for a Glyph Input Is the Full Prev-Output Script

For a Glyph UTXO spend, the scriptCode in the sighash preimage is the **entire 63-byte Glyph-P2PKH scriptPubKey**, not just the 25-byte P2PKH tail:

```python
# WRONG — only the P2PKH portion
redeem_script = bytes.fromhex("76a914<hash>88ac")

# CORRECT — the full Glyph UTXO scriptPubKey
redeem_script = bytes.fromhex("d8<ref36>75" + "76a914<hash>88ac")
```

### Example Transaction

First Ledger-signed Glyph spend on mainnet:

- **Parent (mint)**: `6c32fcbbd6834170b3afcb9bbed759eeb21db72fd509790a3cb804c6eb5c0630:0`
- **Spend tx**: [`22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3`](https://explorer.radiantblockchain.org/tx/22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3)
- **Proof**: Radiant mainnet consensus accepted both the Glyph-input and plain-P2PKH-input signatures from the Ledger

### Privacy Caveat

If you pair your Glyph UTXO with another Ledger UTXO to pay the fee, you prove on-chain that both UTXOs belong to the same wallet. If one input chain traces back to a KYC source and the other to a Glyph mint with separate provenance, combining them **breaks that privacy boundary**. Consider funding a fresh Ledger address from an unrelated source before spending mint-to-Ledger Glyphs.

---

## 7. For Wallet Developers: Integrating Ledger Support

If you maintain a Radiant wallet and want to add Ledger support, here's the checklist.

### 7.1. Detect the Device

SLIP-44 coin type is **512**. Use `m/44'/512'/0'/0/0` as the default receive path. The device's runtime path-lock will refuse derivation outside `m/44'/512'/...`.

```python
DEVICE_IDS = [(0x2c97, 0x5000)]  # Nano S Plus
COIN_TYPE = 512
DEFAULT_PATH = "m/44'/512'/0'/0/0"
```

### 7.2. Construct Transactions

For **plain P2PKH sends** — no special handling needed beyond SLIP-44. Existing Electron Cash / BCH codepaths work. Use `SIGHASH_ALL | SIGHASH_FORKID = 0x41`.

For **Glyph UTXO spending** — teach your script parser to recognize Glyph-tail scripts:

```python
def classify_glyph_p2pkh(script_bytes):
    """Return (address, glyph_refs) or None."""
    if len(script_bytes) < 25:
        return None
    # Skip leading push-ref wrappers: each is 1 opcode byte + 36 bytes
    GLYPH_OPCODES = {0xD0, 0xD1, 0xD2, 0xD3, 0xD8}
    refs = []
    i = 0
    while i < len(script_bytes):
        op = script_bytes[i]
        if op in GLYPH_OPCODES:
            refs.append(script_bytes[i+1:i+37])
            i += 37
            # Skip OP_DROP if present (typical Glyph wrapper)
            if i < len(script_bytes) and script_bytes[i] == 0x75:
                i += 1
        else:
            break
    # Check if remainder is standard P2PKH
    tail = script_bytes[i:]
    if len(tail) == 25 and tail[:3] == b"\x76\xa9\x14" and tail[23:] == b"\x88\xac":
        return (p2pkh_address(tail[3:23]), refs)
    return None
```

Once the wallet links the Glyph UTXO to the owning address, the rest of the send flow (choosing inputs, building outputs, signing) works via standard code.

### 7.3. Sign with the Ledger

Use the `btchip-python` library (vendored into `Electron-Wallet` to work around upstream being archived). The APDU flow for a segwit-style sighash (BCH-family, which Radiant uses):

```python
# Setup: trust all inputs, compute prev/sequence hashes
for utxo in inputs:
    trusted = app.getTrustedInput(prev_tx_obj, utxo.vout)
    trusted["sequence"] = sequence_hex
    trusted["witness"] = True   # CRITICAL — triggers BIP143-style preimage
    chip_inputs.append(trusted)

app.enableAlternate2fa(False)

# First pass: setup all inputs + hash outputs
app.startUntrustedTransaction(True, 0, chip_inputs, first_redeem_script, version=0x02)
output_data = app.finalizeInput(b"", 0, 0, change_path, raw_unsigned_tx)

# Per-input signing — pass ONE input at a time with inputIndex=0
for i in range(len(chip_inputs)):
    app.startUntrustedTransaction(False, 0, [chip_inputs[i]], redeem_scripts[i], version=0x02)
    sig = app.untrustedHashSign(derivation_paths[i], lockTime=0, sighashType=0x41)
    signatures.append(sig[:-1] + b"\x41")
```

### 7.4. Output-Side Restrictions

The Ledger app's output FSM rejects outputs that aren't:

- **Plain P2PKH** (`76a914 <hash> 88ac`, 25 bytes)
- **P2PKH-prefixed Glyph** (`76a914 <hash> 88ac <OP_DISALLOW…> <push-ref> <ref> ...`)

P2SH destinations (`a914 <hash> 87`) and OP_RETURN memos are **not supported in v1.0**. If your send tx has one, the device will return SW 6F0F (`SW_TECHNICAL_PROBLEM_2`).

### 7.5. Reference Implementation

See [`Zyrtnin-org/Electron-Wallet`](https://github.com/Zyrtnin-org/Electron-Wallet/tree/radiant-ledger-512) for a working reference implementation. Key files:

- [`electroncash_plugins/ledger/ledger.py`](https://github.com/Zyrtnin-org/Electron-Wallet/blob/radiant-ledger-512/electroncash_plugins/ledger/ledger.py) — main plugin
- [`electroncash_plugins/ledger/vendor/btchip/`](https://github.com/Zyrtnin-org/Electron-Wallet/tree/radiant-ledger-512/electroncash_plugins/ledger/vendor/btchip) — vendored btchip (upstream archived)

---

## 8. For Researchers: Radiant's Sighash Preimage

Radiant's signature preimage is **NOT byte-identical to BCH**, despite both using `SIGHASH_ALL | SIGHASH_FORKID = 0x41`.

### 8.1. The Extra Field: `hashOutputHashes`

Radiant inserts a new 32-byte field between `nSequence` and `hashOutputs` in the preimage:

```
version              int32 LE    (4 bytes)
hashPrevouts         sha256d     (32 bytes)
hashSequence         sha256d     (32 bytes)
prev_tx_id (LE)      bytes       (32 bytes)
prev_output_index    uint32 LE   (4 bytes)
scriptCode length    varint
scriptCode           bytes
input_satoshis       uint64 LE   (8 bytes)
input_sequence       uint32 LE   (4 bytes)
hashOutputHashes     sha256d     (32 bytes)   ← Radiant's addition
hashOutputs          sha256d     (32 bytes)
nLockTime            uint32 LE   (4 bytes)
sighashType          uint32 LE   (4 bytes)
```

This field is present for **every** Radiant tx, including plain P2PKH. BCH's signing path doesn't produce it, so stock BCH-family Ledger apps produce signatures that Radiant mainnet rejects with `script-execution-error`.

### 8.2. Computing `hashOutputHashes`

`hashOutputHashes` is `sha256d` of the concatenation of per-output 76-byte summaries:

```
nValue                  uint64 LE   (8 bytes)
sha256d(scriptPubKey)   bytes       (32 bytes)
totalRefs               uint32 LE   (4 bytes)
refsHash                bytes       (32 bytes)
```

For plain P2PKH (no push-ref opcodes): `totalRefs = 0`, `refsHash = 32 zero bytes`.

For Glyph outputs (script contains `OP_PUSHINPUTREF` 0xD0 or `OP_PUSHINPUTREFSINGLETON` 0xD8):
- Extract each push-ref opcode's 36-byte payload
- Deduplicate
- Sort lexicographically (by hex)
- Concatenate
- `refsHash = sha256d(concatenated refs)`
- `totalRefs = count of unique refs`

Note: `OP_REQUIREINPUTREF` (0xD1) and `OP_DISALLOWPUSHINPUTREF` (0xD2) do **NOT** contribute to `totalRefs` or `refsHash`. Only `PUSHINPUTREF` and `PUSHINPUTREFSINGLETON` count.

### 8.3. Canonical Source

- [`radiant-node/src/script/interpreter.cpp:2636-2650`](https://github.com/RadiantBlockchain/radiant-node/blob/master/src/script/interpreter.cpp#L2636) — C++ consensus implementation
- [`radiantjs sighash.js:91-237`](https://github.com/RadiantBlockchain/radiantjs/blob/master/lib/transaction/sighash.js#L91) — JS reference implementation
- [`radiant-ledger-app/scripts/radiant_preimage_oracle.py`](https://github.com/Zyrtnin-org/radiant-ledger-app/blob/main/scripts/radiant_preimage_oracle.py) — Python port (self-validated against 5 mainnet fixtures)

### 8.4. Validated Against Mainnet

The `radiant_preimage_oracle.py` is triple-validated:

- **A**: recomputes sighashes for real mainnet txs, verifies published signatures
- **B**: hand-computed byte-diff of a plain P2PKH preimage matches oracle output
- **C**: cross-validates against a second independent signer (FlipperHub's Node.js signing)

Golden fixtures in [`scripts/fixtures/preimage-vectors.json`](https://github.com/Zyrtnin-org/radiant-ledger-app/blob/main/scripts/fixtures/preimage-vectors.json) cover 5 mainnet-confirmed txs: 18 sighashes total, all verify.

### 8.5. Why This Was Missed Initially

The v1 walking-skeleton reached "device signs → mainnet rejects with script-execution-error" before discovering that Radiant's sighash diverges from BCH. The lesson: `SIGHASH_FORKID` with the same sighash-type byte (0x41) doesn't guarantee byte-identical preimage. Consensus-level differences can exist without any changes to the sighash type byte.

For details on the remediation, see [`docs/solutions/integration-issues/radiant-preimage-hashoutputhashes-missing.md`](https://github.com/Zyrtnin-org/radiant-ledger-app/blob/main/docs/solutions/integration-issues/radiant-preimage-hashoutputhashes-missing.md) in the planning repo.

---

## 9. Reproducibility: Build & Verify Yourself

### 9.1. Reproducible Device Build

The Docker image is pinned by digest. Anyone who runs:

```bash
docker run --rm -v "$(pwd):/app" -u "$(id -u):$(id -g)" \
  ghcr.io/ledgerhq/ledger-app-builder/ledger-app-builder-lite@sha256:b82bfff7862d890ea0c931f310ed1e9bce6efe2fac32986a2561aaa08bfc2834 \
  bash -c "cd /app && make COIN=radiant BOLOS_SDK=\$NANOSP_SDK"
```

…should get the same `bin/app.hex` SHA256 as the published release. Cross-check against [`app-radiant/BUILDER.md`](https://github.com/Zyrtnin-org/app-radiant/blob/main/BUILDER.md).

### 9.2. Python Oracle Validation

No hardware required. Re-runs the 3-way validation:

```bash
git clone https://github.com/Zyrtnin-org/radiant-ledger-app.git
cd radiant-ledger-app/scripts

python3 oracle_self_validate.py        # Exit 0 = 3-way PASS
python3 test_oracle_against_vectors.py # Exit 0 = all 5 fixtures PASS
python3 test_push_refs.py              # Exit 0 = 22 push-ref unit tests PASS
```

Pure Python + stdlib + `ecdsa`. Anyone running these on a clean clone gets the same result.

### 9.3. Device-vs-Oracle Tests (Hardware Required)

```bash
python3 test_device_plain_sign.py      # Plain P2PKH sign — sig verifies
python3 test_device_glyph_sign.py      # Glyph output sign — sig verifies
python3 spend_real_glyph_2in.py        # Full Glyph spend — produces broadcastable tx
```

These require:
1. A Nano S Plus with the Radiant app sideloaded
2. A real Ledger-owned UTXO to spend (or a known test UTXO)

### 9.4. SHA256 Cross-Check

Every tagged release publishes `bin/app.sha256`. Example:

```
ee141e5cea504bf71b5624838f6d27223c6b8379555c3366032f9fa91384f00f  app.hex   # v0.0.4
13da09190ed4bd6b73a4f5933eaf689f569912ea1c07540d75ef051974b98800  app.hex   # v0.0.5 initial (Glyph support)
c78a5848680f931f10354690515ae8c9dc56a6d2f41bdfd88f22ca8f1320c258  app.hex   # v0.0.5 (output_script_is_regular relaxed)
51bb1da11b64785d51d80e65594bda94da1d3307177c460de43c3dafe0bddab7  app.hex   # v0.0.6 (security hardening + Radiant icon)
7e51dbff88a42752fdff333ee0f26ace9e740e7381c99737ead1f65da7318f0f  app.hex   # v0.0.7 (FSM bounds + disallow-ref conflict detection)
```

If your build's SHA256 matches, the binary is byte-identical.

---

## 10. Troubleshooting

### Status Codes (SW values returned by the device)

| SW | Name | Meaning | Fix |
|----|------|---------|-----|
| `0x9000` | `OK` | Success | — |
| `0x6982` | `SECURITY_NOT_SATISFIED` | Device locked | Enter PIN |
| `0x6985` | `CONDITIONS_NOT_SATISFIED` | User declined or approval missing | Approve on-device before signing |
| `0x6A80` | `INCORRECT_DATA` | Malformed input (e.g., varint > max script size) | Check your tx structure |
| `0x6D00` | `INS_NOT_SUPPORTED` | Wrong app open on device | Open the Radiant app first |
| `0x6D09` | `INS_NOT_SUPPORTED` (v2) | App not opened | Open the Radiant app first |
| `0x6F00` | `TECHNICAL_PROBLEM` | Internal crypto error | Re-plug, retry |
| `0x6F0F` | `TECHNICAL_PROBLEM_2` | Output script rejected by FSM (e.g., P2SH, OP_RETURN) | Check output shape — v1 supports P2PKH + Glyph-P2PKH only |
| `0x6700` | `WRONG_LENGTH` | APDU malformed | Check BIP32 path format |
| `0x6702` | `?` | Device in dashboard, not Radiant app | Open the Radiant app first |
| `0x5515` | `LOCKED` | Device locked | Enter PIN |

### Common Errors

#### "min relay fee not met" on broadcast

Radiant's min relay is **10,000 sats/byte**. Calculate `tx_size × 10_000` and ensure your fee meets it. For a typical ~225-byte send that's 2,250,000 sats = 0.0225 RXD.

```python
# Rule of thumb
fee_sats = tx_size_bytes * 10_000
```

#### "Device returns SW 6D00 on getWalletPublicKey"

The Radiant app isn't currently open on the device. Unlock the Nano S Plus and launch the Radiant app from the dashboard.

#### "Invalid sequence" or "open failed" on USB

HID interface is in a bad state (common after crashes). Fix:

1. Close any running Electron Radiant processes
2. Unplug + replug the USB cable
3. Unlock + open the Radiant app
4. Retry

#### Balance is wrong (lower than expected) after receiving a Glyph

Electron Radiant doesn't show Glyph UTXOs in the Coins tab (section 6). The balance excludes them. Verify receipt directly on a block explorer.

#### "This app is not genuine" persistent banner

This is **expected and cannot be removed** for community-built Ledger apps without Ledger's review. Not a security issue — if you verified the SHA256 of the build, you know the code is the code.

---

## 11. Appendix: Quick Reference

### BIP32 Paths

```
m/44'/512'/0'/0/0     First receive address
m/44'/512'/0'/0/N     Receive address N
m/44'/512'/0'/1/0     First change address
m/44'/512'/0'/1/N     Change address N
```

The app's runtime path-lock rejects anything outside `m/44'/512'/.../....`.

### Sighash Type

Always `0x41` (`SIGHASH_ALL | SIGHASH_FORKID`) for v1.0. SIGHASH_SINGLE and SIGHASH_ANYONECANPAY are v2 scope.

### Fork Value

Radiant uses fork value **0** (same as BCH). The fork byte isn't in the sighash type byte; it's baked into the sighash preimage computation rules.

### Glyph Opcodes

| Opcode | Hex | Effect on `hashOutputHashes` |
|--------|-----|------------------------------|
| `OP_PUSHINPUTREF` | `0xD0` | Contributes to `refsHash` |
| `OP_REQUIREINPUTREF` | `0xD1` | Does NOT contribute |
| `OP_DISALLOWPUSHINPUTREF` | `0xD2` | Does NOT contribute |
| `OP_DISALLOWPUSHINPUTREFSIBLING` | `0xD3` | Does NOT contribute |
| `OP_PUSHINPUTREFSINGLETON` | `0xD8` | Contributes to `refsHash` |

Each push-ref opcode is followed by exactly 36 bytes of payload (32B txid + 4B vout LE).

### Verified Mainnet Transactions

| Tx | Description | Block |
|----|-------------|-------|
| [`de3574979f…`](https://explorer.radiantblockchain.org/tx/de3574979f986616b4152c4294b85562318292490d3587d8fe32aff456893743) | First Ledger-signed Radiant tx | 420762 |
| [`e10517b534…`](https://explorer.radiantblockchain.org/tx/e10517b534db04d20817a75d8c9522a4046ce167808d46b7b6de2eacf1e5ba9e) | Plain P2PKH regression on v0.0.5 | mempool |
| [`22d4e0e072…`](https://explorer.radiantblockchain.org/tx/22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3) | First Ledger-signed Glyph UTXO spend | mempool |

### Looking for Testers

Have a Nano S Plus and some spare RXD? Open an issue on [`app-radiant`](https://github.com/Zyrtnin-org/app-radiant/issues) or ping on the Radiant Discord to get the pre-release install working on your device. We're validating across firmware versions and tx shapes before wider release.

### Related Reading

- [`radiant-glyph-nft-guide`](https://github.com/Zyrtnin-org/radiant-glyph-nft-guide) — how to **mint** Glyph NFTs (software signing path)
- [`radiant-ledger-app`](https://github.com/Zyrtnin-org/radiant-ledger-app) — planning, Python oracle, investigation notes
- [`app-radiant`](https://github.com/Zyrtnin-org/app-radiant) — the Ledger app source
- [`Electron-Wallet@radiant-ledger-512`](https://github.com/Zyrtnin-org/Electron-Wallet/tree/radiant-ledger-512) — patched Electron Radiant

---

## License

MIT — see [LICENSE](LICENSE).

Referenced upstream projects (LedgerHQ/app-bitcoin, radiantjs, radiant-node) retain their own licenses.
