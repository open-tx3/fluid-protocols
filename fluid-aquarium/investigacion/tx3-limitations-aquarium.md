# TX3 Limitations Found During Fluid Aquarium Implementation

Discovered with trix 0.21.1 (2026-04-08). Updated 2026-04-14 with unreleased fixes.

---

## Resolved

### 1. Param/Field Name Collision (Lowering Panic) — RESOLVED ([tx3#316](https://github.com/tx3-lang/tx3/pull/316))

Same as Indigo limitation #2. Type field names must not collide with tx param names.

**Fix:** Support shadowing of record field names ([tx3#316](https://github.com/tx3-lang/tx3/pull/316)). All type field prefixes (`ct_`, `td_`, `co_`, `st_`, `opf_`, etc.) have been removed. Field names can now match tx param names.

### 2. Withdrawal Redeemer Not Generated in Witness Set — RESOLVED ([tx3#317](https://github.com/tx3-lang/tx3/pull/317))

**Was: Severity High — blocked on-chain execution of `consume_oracle`**

The `cardano::withdrawal` block correctly added the withdrawal entry to the tx body but the corresponding redeemer (purpose=reward, index=0) was **NOT generated** in the witness set.

**Fix:** Compile withdrawal redeemers correctly ([tx3#317](https://github.com/tx3-lang/tx3/pull/317)). Verified with `trix invoke`: the `consume_oracle` tx now generates both the spend redeemer (`ConsumeOracle`, tag=4) and the reward redeemer (`OracleRedeemer::FeedCharlie`) in the witness set. No external CBOR post-processing needed.
---

## Partially Resolved

### 6. Reference Datum Access — PARTIALLY RESOLVED ([tx3#318](https://github.com/tx3-lang/tx3/pull/318))

Same as Indigo limitation #4. Reference blocks add UTxOs to `reference_inputs` but tx3 could not read their datum values.

**Fix:** Support typed access to reference datums ([tx3#318](https://github.com/tx3-lang/tx3/pull/318)). Syntax: `reference name { ref: utxo, datum_is: Type, }` then access `name.field`.

**What works:**
- `datum_is` successfully resolves the UTxO and reads the datum at runtime
- 1-level field access compiles and resolves: `ref.field` where field is a top-level field of the datum type

**What doesn't work:**

1. **Multi-level field access:** `ref.field.subfield` fails at compile time with "not in scope: subfield". Only 1 level of `.field` is supported.

2. **Field accessor resolves to Assets, not datum:** At runtime, `ref.field` resolves the reference to its UTxO value (Assets type), not to the datum. The Assets type does not support property access at all — even index 0 fails. Tested with both `params_data.min_to_stake` (index 0) and `params_data.min_ada` (index 3): both fail with `property index N not found in Assets([...])`. The `datum_is` annotation does not change what the reference resolves to — it always resolves to Assets.

**Tested with Charli3 oracle provider datum** (`b7ba3c4a...#1`):
- The runtime reads the full datum structure: `Struct(constructor: 0, fields: [Struct(constructor: 2, fields: [Map([(0, 270171), (1, 1776135202109), (2, 1776221602109)])])])`
- But cannot extract values: `ref.field` resolves to Assets (not datum), and only 1 level of access is supported

**Impact on Aquarium:**
- `oracle_price`, `oracle_valid_from`, `oracle_valid_to` are in the oracle provider datum but cannot be read — must remain as invoke params
- `params_data.min_ada` (ParamsDatum) cannot be read — `payment_ada` must remain as invoke param
- Oracle feed UTxO (`7f3bb225...#0`) has no datum — just an NFT marker

**What was achieved without `datum_is`:** `consume_oracle` reduced from 21 to 17 params by:
- Moving `oracle_feed_ref` and `oracle_contract_ref` to env (stable deployment UTxOs)
- Reusing `payment_token_policy`/`payment_token_name` for the oracle token fields (protocol invariant: the oracle prices the same token being paid)

With a working `datum_is`, 4 more params could be eliminated: `oracle_price`, `oracle_valid_from`, `oracle_valid_to` (from oracle provider datum) and `payment_ada` (from params datum `min_ada`).

**Proposed enhancements for tx3-lang:**
1. Fix field accessor to read from datum (not UTxO assets) when `datum_is` is specified
2. Support multi-level field access: `ref.field.subfield`

---

## Active Limitations (impact on production use)

### 3. Custom Types Cannot Be Passed as Parameters

Same as quirk #6 in `tx3-quirks.md`. The `Bytes` type wraps values as CBOR ByteString, not raw Plutus Data.

**Impact on Aquarium:**
- `execute_scheduled`: The `batcher` field in the `ScheduledTx` redeemer expects an on-chain `Address` type (Constr with payment/stake credentials). Passing it as `Bytes` produces a ByteString wrapper instead.

**Workaround applied:** Pass `"00"` placeholder for `batcher_addr_cbor`. For production, the caller must construct the full redeemer CBOR externally.

### 4. List Values Cannot Be Passed as Invoke Parameters

`List<T>` works correctly in type definitions and as hardcoded literals (e.g., `[]`). However, list values **cannot be passed as JSON invoke args**. The `from_json()` resolver only supports Int, Bool, Bytes, Address, and UtxoRef.

**Impact on Aquarium:** The `signatures` field of `OracleRedeemer` is `List<OracleSignature>`. Since PriceDataCharlie txs on-chain use 0 signatures, the empty list `[]` is hardcoded in the tx3 source. If a variant required non-empty signatures, they could not be passed dynamically.

**Workaround applied:** Hardcoded `[]` in tx source. Works for the PriceDataCharlie case.

### 5. Signers Extracts Payment Key — Some Validators Need Staking Key

`signers { Party, }` works and extracts the **payment key hash** from the party address. This is correct for `consume_oracle` (signer = user payment key) and `withdraw_tank`.

However, `execute_scheduled` and `stake_fldt` require the **staking key hash** as required signer (the Aiken validators extract it from `address.stake_credential`). tx3 has no way to extract the staking key from an address.

**Workaround applied:** `consume_oracle` and `withdraw_tank` use `signers { User, }`. `execute_scheduled` and `stake_fldt` use a raw `signer_hash: Bytes` param with the staking key hash.

**Proposed enhancement:** Support `signers { stake_key_of(Party), }` or similar to extract the staking credential.

### 7. Collateral Requires Pure ADA UTxO

The `collateral {}` block resolves a UTxO from the specified party's address. Cardano requires collateral UTxOs to contain **only ADA** (no native tokens). If the wallet's UTxOs all contain native tokens, collateral resolution fails with "Input not resolved".

**Workaround applied:** Use wallets that have at least one pure-ADA UTxO.

---

## Not Applicable / Non-Issues

### Metadata

On-chain Aquarium txs contain `CBORTag(259, {})` as auxiliary data — an empty CIP-68 metadata tag. Not protocol-specific, not modeled.

### Redeemer Index Differences

Generated redeemer indices may differ from on-chain because input ordering depends on UTxO selection at build time. Expected and correct.

---

## Summary Table

| # | Limitation | Severity | Status | Workaround |
|---|-----------|----------|--------|------------|
| 1 | ~~Param/field name collision~~ | ~~Low~~ | **Resolved** ([tx3#316](https://github.com/tx3-lang/tx3/pull/316)) | ~~Prefix type fields~~ |
| 2 | ~~Withdrawal redeemer not in witness set~~ | ~~High~~ | **Resolved** ([tx3#317](https://github.com/tx3-lang/tx3/pull/317)) | ~~Post-process CBOR externally~~ |
| 3 | Custom types as params (Address in redeemer) | Medium | Active | Bytes placeholder |
| 4 | List values not passable as invoke args | Low | Active | Hardcode in tx source |
| 5 | Signers require raw key hash | Low | Active | Pass hash as Bytes param |
| 6 | Ref input datum access | Medium | **Partial** ([tx3#318](https://github.com/tx3-lang/tx3/pull/318), accessor bug + no nesting) | Pass values as params |
| 7 | Collateral needs pure ADA UTxO | Low | Active | Use correct wallet |
