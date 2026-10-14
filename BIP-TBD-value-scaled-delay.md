---
BIP: TBD
Layer: Consensus (soft fork)
Title: Value-Scaled Spend Delay (VSD) for Large-Value UTXOs
Authors: (draft author, pending)
Comments-Summary: No comments yet.
Comments-URI: TBD
Status: Draft
Type: Standards Track
Created: 2026-10-14
License: BSD-3-Clause
Requires: TBD
---

# Value-Scaled Spend Delay (VSD)

## Abstract

This proposal introduces a single consensus rule: **spending a confirmed UTXO whose value meets or exceeds a threshold requires a relative height delay that scales with the UTXO's value.** Small transactions are unaffected. The rule is a pure soft fork (it only *reduces* the set of valid transactions), changes no data format, adds no new cryptographic primitive, and introduces no new opcode. Its purpose is to convert the confirmation of a large-value transfer from "within one block (~10 minutes)" into "after N blocks", thereby giving a legitimate holder a discovery-and-response window if a key is compromised — without disrupting day-to-day transaction flow.

This proposal is intentionally **partial and conservative**. It is not a substitute for quantum-resistant signatures (e.g. a future P2MR/PQ-signature soft fork) or for public-key-hiding output types (P2WPKH/P2TR). It is an additive, behavior-layer defense intended to buy time in the common, recoverable scenario: *a key is stolen or broken, but the holder is alive and monitoring.*

---

## Rationale

### The gap

Bitcoin UTXOs are non-revocable. A "cancellation" of a transfer is not a real revoke; it is a *competing* spend of the same UTXO, funded by the same private key. When a private key is compromised (stolen, or derived by a cryptographically relevant quantum computer from an exposed public key), the attacker holds the same key as the victim. A transfer of a large UTXO confirms in ~10 minutes, which is typically too fast for the victim to notice and respond.

The core value a "cooling-off" / "冷静期" mechanism must buy is **time between the theft attempt and its irrevocable confirmation**. Everything else is secondary.

### Why this shape

- The primitive needed already exists and is battle-tested: **relative height locking (OP_CSV, BIP-68)**, active since 2017. What this BIP adds is not the primitive but a **mandatory, value-scaled application** of it at the consensus layer — so it cannot be opted out of by a sender who simply sets `nSequence = 0`.
- It is a **soft fork**: the new validity set is a strict subset of the old. Old nodes continue to validate the chain new nodes build; they merely do not understand that certain large-value spends were deliberately not included.
- It **only changes behavior logic**, per the design constraint: no new address type, no new output format, no new signature algorithm, no governance layer. The only new thing is one conditional rule keyed on a UTXO's confirmed value.
- It **does not disrupt normal transactions**: below a threshold the delay is zero.

### Threat model (in and out of scope)

| Scenario | VSD helps | Why |
|----------|:---------:|-----|
| Key stolen/broken, holder **online & monitoring** | ✅ primary target | holder observes the pending large spend in mempool and wins the fee race to re-spend |
| Key stolen/broken, holder **offline** | ⚠️ partial | attacker must wait the full delay; VSD at least prevents instant irrevocable theft |
| **No holder** (e.g. lost keys / Satoshi P2PK) | ⚠️ observation only | no one can race; VSD only lengthens the window for the community to observe |
| HD wallet (one key → many addresses) | ⚠️ | racing one UTXO is not enough; the holder must rotate the whole key chain |
| Custodial / exchange-held coins | ⚠️ | depends on the custodian upgrading and monitoring |

VSD is designed to be **honest** about this: it is a "best-effort" defense, a last time window layered *under* the two root-cause fixes (key-hiding outputs and post-quantum signatures). It is not claimed to be a complete solution.

---

## Specification

### Definitions

- **Value.** The number of satoshis in a UTXO's `value` field (internal unit: satoshis; 1 BTC = 100,000,000 sats).
- **`outpoint_height(O)`** — the height of the block that confirmed UTXO `O` (the block containing the creating transaction). For an unconfirmed (mempool-only) previous output, the outpoint is not a confirmed UTXO and is *not* subject to this rule (see "Unconfirmed previous outputs").
- **Threshold `T`** — the minimum UTXO value (in satoshis) at which the delay applies. A UTXO with `value < T` is not subject to the delay.
- **Delay function `D(value)`** — a monotonic, non-decreasing step function mapping a UTXO value to a delay in number of blocks. See table below.
- **Spending height `H`** — the height of the block that includes the spending transaction (for block validation), or the chain tip height (for mempool eligibility).

### The rule

For each transaction input referencing a **confirmed** previous output `O`:

```
if O.value >= T:
    required = outpoint_height(O) + D(O.value) + 1
    the spending transaction may only be included in a block of height H
    such that H >= required.
```

`required` is the **earliest block height at which a spend of the UTXO is mineable**. The `+1` is not a VSD invention — it is the standard Bitcoin rule that a UTXO cannot be spent in the same block in which it is confirmed. For a zero-delay (`D=0`) UTXO this simply yields the normal "spendable next block" behavior; for a positive `D` it yields `D` blocks *after* confirmation.

In other words: this is **exactly the BIP-68 relative-height evaluation** — a `D`-block relative height lock on an outpoint confirmed at height `c` is satisfied when the reference height (the height of the block *before* the spending block, i.e. `H-1`) satisfies `H-1 >= c + D`, i.e. `H >= c + D + 1`. VSD makes this **mandatory and value-keyed** at the consensus layer rather than opt-in via the `nSequence` field. The `nSequence` field is **not** consulted for this check: even a sender who sets `nSequence = 0` on every input cannot bypass the delay on a large-value UTXO.

### Delay function (proposed starting values — tunable, to be finalized per BIP-2 process)

| UTXO value | Delay `D(value)` | Approx. time |
|-----------:|:---------------:|:------------:|
| `< 1 BTC` (value < 100,000,000 sats) | 0 | none (no impact) |
| `1 – 10 BTC` | 144 | ~1 day |
| `10 – 100 BTC` | 4320 | ~3 days |
| `100 – 1000 BTC` | 10080 | ~7 days |
| `>= 1000 BTC` | 20160 | ~14 days |

These are **draft values** for discussion. The exact thresholds and delays are consensus parameters (see "Open parameters") and should be conservative at activation: the goal is "large-only, moderate delay" at first, letting the economy self-adjust.

Notes on the function:
- **Monotonic non-decreasing**: a larger UTXO never gets a shorter delay.
- **Per-UTXO, keyed on the spent output's value** (not the input amount, not the sum of inputs, not the output values of the new transaction).
- **Delay travels with value.** To move a large UTXO you must spend it; that spend references the large UTXO and is therefore delayed. You cannot launder a 1000 BTC UTXO into ten 100 BTC UTXOs without the 1000 BTC spend already incurring the delay.

### Reference height: block vs. mempool

- **Block validation:** `H` is the height of the block being validated. If any input fails the rule, the block is rejected.
- **Mempool:** `H` is the current chain tip height. A transaction whose any input is not yet "matured" (tip height < required) is **held in the mempool as non-mineable** — it is accepted for relay/storage but not eligible for inclusion until the tip reaches the required height. Once matured, it becomes includable normally (subject to fee and block space).

### Unconfirmed previous outputs

If the referenced previous output is not yet confirmed (i.e. the transaction is a child of an in-mempool parent, as in CPFP), the input is **not** subject to the value-delay, because `outpoint_height` is undefined for an unconfirmed UTXO. The value-delay is a property of *confirmed* large UTXOs: once such a UTXO is confirmed, any later spend of it is delayed. A child of an unconfirmed large-value parent inherits the parent's eventual constraints transitively (the parent, when it is confirmed, will itself have been subject to the delay).

### Validation pseudocode

```
function input_is_pending_vsd(input, utxo_set, ref_height):
    # input.prevout = (txid, vout)
    prev = utxo_set.lookup(input.prevout)
    if prev is None:
        return false            # not a confirmed UTXO -> not subject to VSD
    if prev.value < T:
        return false            # below threshold -> no delay
    required = prev.outpoint_height + D(prev.value) + 1   # earliest mineable height
    return ref_height < required   # True => spend not yet allowed

# Block validation: for every tx in block, for every input:
#     if input_is_pending_vsd(input, utxo_set, block_height): reject block

# Mempool admission: a tx is eligible for mining only if, for all inputs:
#     not input_is_pending_vsd(input, utxo_set, tip_height)
```

### Interaction with existing mechanics

- **RBF (BIP-125):** Compatible. Competing replacements reference the same UTXO and share the same `required` height; you may re-sign and replace during the window, but no replacement confirms before `required`.
- **CPFP / child-pays-for-parent:** Compatible. A child can pay the fee of a pending large-value parent, but the pair still cannot confirm before the parent's `required` height.
- **Fee market:** Fee no longer guarantees *early* confirmation for large-value UTXOs — it only affects inclusion order after maturation. This is the intended property: price buys priority *within* the window, not the *start* of the window.
- **Floor, not ceiling:** VSD is the *minimum* delay enforced by consensus. Holders may voluntarily impose a **longer** delay or additional spending conditions on their own UTXOs via existing `OP_CSV`/`OP_CLTV` and Taproot scripts. VSD never restricts a holder from making their own coins *harder* to spend than the rule requires.

### Grandfathering / activation behavior

The rule applies **only to UTXOs created at or after the activation block height**. Existing (pre-activation) UTXOs retain their current spend behavior for as long as they remain unspent.

Rationale: a soft fork must not retroactively change how the *existing* UTXO set is spent, or it would "freeze" legitimate cold holdings on a flag day with no warning — a far more disruptive behavior change. As the existing stock gets recycled (every confirmed spend re-creates its outputs post-activation), the protection **ratchets in gradually** over time.

Acknowledged gap: an *existing* large pre-activation UTXO can still be spent immediately by a key-holder. This is accepted as the standard soft-fork conservatism tradeoff; closing it fully would require a one-time consensus migration of the UTXO set, which this proposal deliberately avoids.

---

## Deployment

Deployed as a **soft fork via BIP-9 version bits** (current version-bits implementation of BIP-9).

### Version-bit assignment

The `version` field is a 32-bit integer; for version-bits signaling the base is `0x20000000` and signal bits occupy the low bits (bits 28–31 are reserved for the version number). On mainnet, **bit 0** has been used (BIP-68 CSV / BIP-141 SegWit) and **bit 1** has been used (BIP-341/342/343 Taproot). This deployment uses the **lowest unused bit — bit 2**:

| bit | version value | prior mainnet use |
|-----|---------------|-------------------|
| 0 | `0x20000001` | BIP-68, BIP-141 |
| 1 | `0x20000002` | BIP-341/342/343 (Taproot) |
| **2** | **`0x20000004`** | **this deployment (VSD)** |
| 3 | `0x20000008` | (free) |

### BIP-9 parameters (draft)

| Parameter | Value | Note |
|-----------|-------|------|
| deployment name | `vsd` | internal identifier |
| version bit | **2** | signal `0x20000004` |
| `nWindow` | **2016** | 1 week of 10-min blocks (standard) |
| `nThreshold` | **1916** | 95% of window (standard) |
| `nStart` | **TBD** | Unix epoch set only after signet + testnet4 validation completes |
| `nTimeout` | **TBD** | `nStart` + 31,557,600 s (1 Julian year) by default |
| `min_activation_height` | derived | per BIP-9: the first 2016-block boundary ≥ `nStart` at which the threshold can first be met |

**Concrete draft shape** (epoch values are placeholders pending review + testnet completion; the *structure* is fixed):
- `nStart` — a date ~12–18 months out, chosen so at least one full signal window elapses after it before the first eligible boundary.
- `nTimeout` = `nStart` + 31,557,600 s.
- If the 1916-threshold is met within a window whose start ≥ `nStart` and < `nTimeout`, the deployment activates at the **next 2016-block boundary**. If not met by `nTimeout`, it locks out (fail-safe, per BIP-9).

**Consensus activation (this rule):** enforced only for blocks at/after the BIP-9 activation block, and (per grandfathering) only for UTXOs created at/after that block. Non-upgraded nodes treat VSD-constrained large-value spends as valid; upgraded nodes treat early spends as invalid. The chain built by the upgraded majority remains valid to non-upgraded nodes, preserving the soft-fork property.

**Policy/relay:** the mempool "hold-not-mineable" behavior (accept + store, but not include in a block until matured) is a **policy** matter, not part of the activation consensus. Default policy: accept and hold; do not relay as mineable until matured.

**No block-size or data-format change.** The rule consumes no extra bytes; it only references the existing `value` and `outpoint_height` already present in the UTXO set.

### Deployment path

1. **Signet / testnet4 first** — full validation on a developer signet (BIP-325-style) and testnet4, with mining, wallet (maturity-countdown UI), and exchange monitoring exercised against the §Test Vectors. No mainnet commitment yet.
2. **Reference implementation** — BIP-2-compliant patch to Bitcoin Core (consensus maturity check in block/tx validation + mempool hold-not-mineable policy) plus the §Test Vectors as unit/regression tests.
3. **Mainnet via BIP-9** — signal `0x20000004`, window 2016, threshold 1916, `nStart`/`nTimeout` as above. Grandfathering means only UTXOs created at/after the activation block are constrained, so activation is low-disruption.

Activation should follow the standard BIP-2 / BIP-127 process: draft → community review → Core implementation + test vectors → deployment BIP with final `nStart`/`nTimeout`.

---

## Test Vectors

Test vectors below exercise the value-delay rule (Specification §rule + §delay function). All values use the **draft** threshold `T = 1 BTC = 100,000,000 sats` and the **draft** delay table. Heights are absolute chain heights. Per the rule, the earliest mineable height is `required = outpoint_height + D(value) + 1` (the `+1` encodes "cannot spend the confirmation block itself"). A spend is includable **at** `required` and any later height.

### Delay table (reference)

| UTXO value (BTC) | sats | `D(value)` (blocks) |
|------------------:|------:|:-------------------:|
| < 1 | < 100,000,000 | 0 |
| 1 – < 10 | 100,000,000 – 999,999,999 | 144 |
| 10 – < 100 | 1,000,000,000 – 9,999,999,999 | 4320 |
| 100 – < 1000 | 10,000,000,000 – 99,999,999,999 | 10080 |
| >= 1000 | >= 100,000,000,000 | 20160 |

### Boundary & functional cases

| # | UTXO value | outpoint height | `D(value)` | `required` = outpoint+D+1 | spend at (H) | includable? | boundary note |
|--:|-----------:|:---------------:|:----------:|:-------------------------:|:------------:|:-----------:|---------------|
| 1 | 0.5 BTC (50,000,000) | 800,000 | 0 | 800,001 | 800,001 | ✅ yes | **below threshold T** — only the standard +1 applies; spendable next block, no VSD delay |
| 2 | 0.99999999 BTC (99,999,999) | 800,000 | 0 | 800,001 | 800,001 | ✅ yes | **just under T** (1 sat below 1 BTC) — still no VSD delay; confirms boundary is exactly at 1 BTC |
| 3 | 1 BTC (100,000,000) | 800,000 | 144 | 800,145 | 800,001 | ❌ no | **exactly at T (lower bound)** — VSD engages; a spend at the normal next-block (800,001) is now rejected |
| 4 | 1 BTC (100,000,000) | 800,000 | 144 | 800,145 | 800,144 | ❌ no | 1 block **before** maturity — not mineable (mempool-held) |
| 5 | 1 BTC (100,000,000) | 800,000 | 144 | 800,145 | 800,145 | ✅ yes | **maturity boundary** — includable exactly at `required` |
| 6 | 10 BTC (1,000,000,000) | 800,000 | 4320 | 804,321 | 804,320 | ❌ no | **1→10 tier boundary (lower bound of 10-BTC tier)**, 1 block early |
| 7 | 10 BTC (1,000,000,000) | 800,000 | 4320 | 804,321 | 804,321 | ✅ yes | maturity boundary for 10 BTC tier |
| 8 | 9.99999999 BTC (999,999,999) | 800,000 | 144 | 800,145 | 800,145 | ✅ yes | **just under 10-BTC tier** (1 sat below) — stays in 144-block tier, not 4320 |
| 9 | 100 BTC (10,000,000,000) | 800,000 | 10080 | 810,081 | 810,081 | ✅ yes | **10→100 tier boundary (lower bound of 100-BTC tier)**, at maturity |
| 10 | 1000 BTC (100,000,000,000) | 800,000 | 20160 | 820,161 | 820,161 | ✅ yes | **>=1000 top tier**, maturity boundary |
| 11 | 5000 BTC (500,000,000,000) | 800,000 | 20160 | 820,161 | 820,161 | ✅ yes | **top tier saturated** — delay equals the 1000-BTC case; does not grow beyond 20160 |

### Grandfathering cases (activation at height `A`)

These verify the rule applies **only** to UTXOs created at/after the activation block height `A`. Assume `A = 900,000`.

| # | UTXO created at height | UTXO value | `required` | spend H | subject to VSD? | includable? | note |
|--:|:----------------------:|-----------:|:----------:|:-------:|:---------------:|:-----------:|------|
| G1 | 850,000 (< A) | 100 BTC | n/a (grandfathered) | 850,001 | ❌ no | ✅ yes | **pre-activation UTXO** — exempt from VSD; spendable at the normal next block despite value |
| G2 | 900,000 (= A) | 100 BTC | 910,081 (=900,000+10,080+1) | 900,001 | ✅ yes | ❌ no | **created exactly at activation** — subject to VSD (creation-height bound is inclusive); a normal next-block spend is rejected |
| G3 | 900,001 (> A) | 100 BTC | 910,082 (=900,001+10,080+1) | 910,082 | ✅ yes | ✅ yes | **created just after activation**; the delay is keyed on the UTXO's own creation height (900,001), not the activation height. A spend at 910,081 would be ❌ not yet includable. |

### Unconfirmed-previous-output case

| # | prev out | spend H | includable? | note |
|--:|----------|:-------:|:-----------:|------|
| U1 | unconfirmed 50 BTC (in mempool) | tip 100,000 | ✅ yes | unconfirmed prev → not subject to VSD; child can be included (CPFP) |
| U2 | unconfirmed 50 BTC parent; parent later confirms at 100,050 | child re-spent at 100,051 | governed by parent's VSD | the *confirmed* 50 BTC UTXO, when spent, is subject to D(50 BTC)=0; larger confirmed values follow the table |

### Expected wallet/UI behavior (non-consensus, informational)

- A wallet sending from a 10 BTC UTXO confirms at height 800,000 should display a **maturity countdown**: "spendable in 4,321 blocks (~3 days); target height 804,321" (804,321 = 800,000 + 4,320 + 1).
- A 0.5 BTC UTXO should show **no countdown** (below threshold).
- The countdown must be driven by the **UTXO's own outpoint height and value**, independent of `nSequence`.

### Invariants the vectors must catch

1. **Threshold is value-keyed**: cases 1/2 (below T) vs 3 (at T) confirm the boundary sits at exactly 1 BTC — 1 sat below is only the standard next-block rule; 1 sat at/above triggers the 144-block VSD delay.
2. **Tier boundaries are lower-inclusive**: cases 3 (1 BTC), 6 (10 BTC), 9 (100 BTC), 10 (1000 BTC) confirm each tier's delay applies from its lower value bound; case 8 (9.99999999 BTC) confirms the value just *under* a tier keeps the lower delay.
3. **Monotonic & saturated**: cases 10 (1000 BTC) and 11 (5000 BTC) confirm delay equals the top-tier value and does not grow beyond 20160.
4. **Maturity is inclusive**: cases 4 (1 block early → rejected) and 5 (exactly at `required` → accepted) confirm the boundary is `H >= required`, not `H > required`.
5. **Grandfathering is height-keyed**: G1 vs G2 confirm pre-activation UTXOs are exempt, activation-height UTXOs are not.
6. **Unconfirmed prevs are exempt**: U1 confirms an in-mempool large-value UTXO is not VSD-gated until confirmed.

These vectors are intended as the Core `functional`/`unit` test suite for the consensus check in §Validation pseudocode.

---

## Backward compatibility

- **Soft fork (backward-compatible to old nodes).** The new validity set ⊆ old validity set. Old nodes accept the chain; they simply do not relay/mine certain early large-value spends. Standard BIP9 soft-fork caveats apply (old nodes cannot distinguish "invalid under new rules" from "not included"; the upgraded majority governs consensus).
- **No data migration.** Existing UTXOs, addresses, and wallet files are untouched (grandfathering).
- **No client-format change.** `value` and outpoint metadata already exist in the UTXO set.
- **Held transactions:** wallets must handle "large-value tx accepted but pending maturation" gracefully (show a maturity countdown), analogous to existing CSV-maturity display.

---

## Known limitations and non-goals

This proposal is deliberately scoped to *partially* improve security *without* disruptive changes. It **does not** and is **not intended to**:

1. **Prevent theft from a compromised key.** It lengthens the window and enables the holder to race; it does not make the theft impossible. An offline or unaware holder can still lose funds after the delay.
2. **Protect addresses with no living holder** (lost keys, Satoshi's P2PK). For these, VSD only buys *observation* time for the community. Genuine protection there would require a separate community-emergency mechanism (out of scope; that is a governance/centralization-sensitive change this proposal avoids).
3. **Replace root-cause fixes.** Key-hiding outputs (P2WPKH/P2TR/P2MR) and post-quantum signatures remain the primary defenses. VSD is an additive behavior-layer time window layered beneath them.
4. **Handle HD-wallet key chains.** Racing one UTXO does not secure the rest of a key; key rotation is still required.
5. **Be tamper-proof against an attacker who simply waits.** If the attacker waits out the delay and the holder never responds, the theft completes. The delay converts "instant" into "bounded-time", not into "impossible".

Non-goals: no new address type, no new output format, no new signature scheme, no new opcode, no on-chain governance, no change to the data model, no change to block size.

---

## Open parameters

To be finalized during review (each is a tunable consensus/policy parameter):

- `T` — activation threshold (draft: 1 BTC).
- The delay table `D(value)` — tier boundaries and per-tier block counts.
- Whether the delay should be **linear in log(value)** rather than a step function (a continuous alternative).
- Grandfathering: strictly "created at/after activation" vs. a bounded transition window for pre-activation UTXOs.
- Whether a **maximum** delay cap is needed for extreme values (e.g. > 10,000 BTC) to avoid indefinite lock.
- Mempool policy defaults (hold vs. reject) and whether to surface a standard "maturity height" field to relays.

---

## References

- BIP-68: Relative lock-time (OP_CSV) — the primitive this builds on.
- BIP-65: CheckLockTimeVerify (OP_CLTV) — absolute lock-time, related semantics.
- BIP-141/143: SegWit — UTXO/witness set and outpoint metadata.
- BIP-341/342/343: Taproot — existing key-hiding and script-path outputs.
- BIP-9: Version bits-based consensus deployment.
- BIP-125: BIP125-replaceability signaling (RBF).
- BIP-360 (Pay-to-Merkle-Root) / BIP-361 (Post Quantum Migration and Legacy Signature Sunset) — the root-cause quantum-defense layers this proposal complements.
- BIP-2: BIP process and authorship guidelines.

---

## Copyright

This BIP is licensed under the Creative Commons CC0 1.0 Universal Public Domain Dedication.
