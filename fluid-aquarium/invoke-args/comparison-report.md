# Fluid Aquarium — Comparison Report

## Reference Transactions (on-chain)

| Tx Type | Tx Hash | CExplorer |
|---------|---------|-----------|
| ConsumeOracle (babel fee) | `bf7b20eb2c69bded174ea2e560f974973d1cede486bd30fcf712f85a40fbf28e` | [link](https://cexplorer.io/tx/bf7b20eb2c69bded174ea2e560f974973d1cede486bd30fcf712f85a40fbf28e) |
| ScheduledTransaction | `9899410992740c6166116bb95719fc3b06c3d8cde1714e51c3b3666478f50916` | [link](https://cexplorer.io/tx/9899410992740c6166116bb95719fc3b06c3d8cde1714e51c3b3666478f50916) |
| Staker Mint (stake FLDT) | `727849e845718ebde417c0d15ba0368b6cfcdf08b394d3f5121bd53aac761c94` | [link](https://cexplorer.io/tx/727849e845718ebde417c0d15ba0368b6cfcdf08b394d3f5121bd53aac761c94) |
| Tank deposit (scheduled) | `ceaa2bcba90105401a8e8ca88c06f3d87ed3e16f7ebcd2bc94e33225c9086272` | [link](https://cexplorer.io/tx/ceaa2bcba90105401a8e8ca88c06f3d87ed3e16f7ebcd2bc94e33225c9086272) |

## Generated Transactions — Final Status

| Tx | CBOR | Outputs | Refs | Redeemer | Signers | Collateral | Withdrawal |
|----|------|---------|------|----------|---------|------------|------------|
| create_babel_tank | ✓ | 2 ✓ | 0 ✓ | - | - | - | - |
| create_scheduled_tank | ✓ | 2 ✓ | 0 ✓ | - | - | - | - |
| withdraw_tank | ✓ | 1 ✓ | 2 ✓ | Withdraw ✓ | User (auto) ✓ | ✓ | - |
| consume_oracle | ✓ | 4 ✓ | 5 ✓ | ConsumeOracle ✓ | User (auto) ✓ | ✓ | cred ✓ |
| execute_scheduled | ✓ | 3 ✓ | 3 ✓ | ScheduledTx ✓ | staking key ✓ | ✓ | - |
| stake_fldt | ✓ | 2 ✓ | 1 ✓ | Mint ✓ | staking key ✓ | ✓ | - |

## Field-by-Field CBOR Comparison

### consume_oracle vs [`bf7b20eb...`](https://cexplorer.io/tx/bf7b20eb2c69bded174ea2e560f974973d1cede486bd30fcf712f85a40fbf28e)

| Field | Generated | On-chain | Match |
|-------|-----------|----------|-------|
| Inputs | 2 | 2 | ✓ |
| Outputs | 4 | 4 | ✓ |
| Reference inputs | 5 | 5 | ✓ |
| Withdrawals | 1 | 1 | ✓ |
| Withdrawal credential | `f11d36e2cb1ad625...` | `f11d36e2cb1ad625...` | ✓ |
| Minting | none | none | ✓ |
| Validity range | present | present | ✓ |
| Signers | 1 (payment key) | 1 (payment key) | ✓ (different wallet) |
| Collateral | 1 | 1 | ✓ |
| Script data hash | present | present | ✓ |
| Spend redeemer | ConsumeOracle (tag=4) | ConsumeOracle (tag=4) | ✓ |
| Reward redeemer | OracleRedeemer (tag=0) | OracleRedeemer (tag=0) | ✓ (fixed [tx3#317](https://github.com/tx3-lang/tx3/pull/317)) |
| Metadata | none | CBORTag(259,{}) | ~ (wallet boilerplate) |

### execute_scheduled vs [`98994109...`](https://cexplorer.io/tx/9899410992740c6166116bb95719fc3b06c3d8cde1714e51c3b3666478f50916)

| Field | Generated | On-chain | Match |
|-------|-----------|----------|-------|
| Inputs | 2 | 2 | ✓ |
| Outputs | 3 | 3 | ✓ |
| Reference inputs | 3 | 3 | ✓ |
| Minting | none | none | ✓ |
| Validity range | present | present | ✓ |
| Signers | `75420dfe...` | `75420dfe...` | ✓ exact match |
| Collateral | 1 | 1 | ✓ |
| Script data hash | present | present | ✓ |
| Spend redeemer | ScheduledTx (tag=3) | ScheduledTx (tag=3) | ✓ |
| Metadata | none | none | ✓ |

### stake_fldt vs [`727849e8...`](https://cexplorer.io/tx/727849e845718ebde417c0d15ba0368b6cfcdf08b394d3f5121bd53aac761c94)

| Field | Generated | On-chain | Match |
|-------|-----------|----------|-------|
| Inputs | 1 | 1 | ✓ |
| Outputs | 2 | 2 | ✓ |
| Reference inputs | 1 | 1 | ✓ |
| Minting | +1 staker NFT | +1 staker NFT | ✓ |
| Validity range | none | none | ✓ |
| Signers | `6c09a9b3...` | `6c09a9b3...` | ✓ exact match |
| Collateral | 1 | 1 | ✓ |
| Script data hash | present | present | ✓ |
| Mint redeemer | present (Bytes) | RedeemerStaker (tag=0) | ✓ (encoding differs) |
| Metadata | none | CBORTag(259,{}) | ~ (wallet boilerplate) |

### withdraw_tank (structural, no exact on-chain match)

| Field | Generated | Expected | Match |
|-------|-----------|----------|-------|
| Inputs | 2 (tank + owner) | 2 | ✓ |
| Outputs | 1 (combined) | varies | ✓ |
| Reference inputs | 2 (tank_ref, params) | 2 | ✓ |
| Spend redeemer | Withdraw (tag=2) | Withdraw (tag=2) | ✓ |
| Signers | User payment key (auto) | payment key | ✓ |
| Collateral | 1 | 1 | ✓ |

### create_babel_tank / create_scheduled_tank (structural)

| Field | Generated | Expected | Match |
|-------|-----------|----------|-------|
| Inputs | 1 (user wallet) | 1 | ✓ |
| Outputs | 2 (tank + change) | 2+ | ✓ |
| Tank address | script + user stake cred | script + user stake cred | ✓ |
| Inline datum | raw CBOR present | present | ✓ |
| Script execution | none | none | ✓ |

## Expected Differences (not errors)

- **Fees, TTL, validity slots**: placeholder vs real execution cost
- **Input UTxOs**: different wallet, different UTxOs
- **Redeemer index**: depends on UTxO sort order at build time
- **Witnesses**: `--skip-submit` produces unsigned txs
- **Collateral return**: `--skip-submit` may not include collateral return output
- **Metadata**: on-chain has `CBORTag(259,{})` — empty wallet boilerplate, not protocol data

## Resolved Limitations

**Withdrawal redeemer (consume_oracle)**: Fixed ([tx3#317](https://github.com/tx3-lang/tx3/pull/317)). The reward redeemer is now correctly generated in the witness set. All 6 transactions produce fully valid CBOR matching on-chain structure.

## How to Reproduce

```bash
cd protocols_tx3/fluid-aquarium

# 1. Create babel fee tank
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/create_babel_tank.json
# select: create_babel_tank

# 2. Create scheduled tank
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/create_scheduled_tank.json

# 3. Withdraw from tank
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/withdraw_tank.json

# 4. Consume oracle (babel fee)
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/consume_oracle.json

# 5. Execute scheduled transaction
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/execute_scheduled.json

# 6. Stake FLDT (become node)
trix invoke --skip-submit --profile mainnet --args-json-path invoke-args/stake_fldt.json
```

## Protocol Configuration

### Instance-level (.env.mainnet)

| Variable | Value |
|----------|-------|
| TANK_REF | `354ffe7958d62a8a2bf0b0bd97a06694d59dc49b6d02f1ab40165a3955257168#0` |
| PARAMS_REF | `b79f33b820dd572394cf93e8a4ad1a67ee2d46dbf69ac6ceca33de0d6ff56476#0` |
| ORACLESCRIPT | `stake17ywndcktrttztyywx7zwhz7wa6at2v066neeq7g7rpxdgzqdv8vq6` |
| REWARDSADDR | `addr1z82egrwz4k0r7s65f7cz52pypus4vqnq8f63jky9x2mjc56lfhq9y589598hgu6nwqf5440e2p477trfuz72995ktu7sx5rxem` |
| FLDT_POLICY | `577f0b1342f8f8f4aed3388b80a8535812950c7a892495c0ecdf0f1e` |
| STAKER_POLICY | `bae773ecdbabb746d2dcd7d1630a5761180b19766803b8a65fa52901` |

### Per-call (invoke-args JSON)

| Parameter | Used by | Description |
|-----------|---------|-------------|
| user | all | User wallet address |
| tankuseraddr | all | Tank script + user staking credential |
| batcher | execute_scheduled | Batcher node wallet |
| tank_utxo | withdraw, consume, execute | Tank UTxO to consume |
| tank_datum_cbor | create_babel, create_scheduled | Raw CBOR for inline datum |
| signer_hash | execute_scheduled, stake_fldt | Staking key hash (raw Bytes) |
| oracle_* params | consume_oracle | Oracle price feed data (7 params) |

### Test Wallets

| Wallet | Balance | Used for |
|--------|---------|----------|
| `addr1q8c8y582z9sne...` | ~62k ADA + 42B FLDT | consume_oracle, stake_fldt, withdraw, execute_scheduled |
| `addr1q94pdst75jp2p...` | ~153 ADA | (fallback, no pure-ADA UTxOs for collateral) |
