# Fluid Aquarium — On-Chain Research

## Protocol Overview

Aquarium by FluidTokens is a decentralized protocol on Cardano that enables:
- **Babel Fees**: Users pay tx fees with native tokens instead of ADA
- **Scheduled Transactions**: Users pre-fund transactions to be executed at a future time by batcher nodes
- **Node Staking**: Operators stake FLDT to become batcher nodes

Source: https://github.com/FluidTokens/ft-cardano-aquarium-sc (Aiken, PlutusV3)
Node: https://github.com/FluidTokens/ft-aquarium-node (Java, Yaci Store)

## Deployed Contracts (Mainnet)

| Contract | Script Hash | UTxOs |
|----------|-------------|-------|
| Tank | `f9724c47299e745cb4f50f9d36cbbadcdf87e015a9d99d927dc4e866` | 808+ |
| Staker | `bae773ecdbabb746d2dcd7d1630a5761180b19766803b8a65fa52901` | 39 active |
| Parameters | `f0e403df77b2bfee7c0799ef927b3763165033cbe38bddc802934883` | 1 |
| Oracle | `1d36e2cb1ad625908e3784eb8bceeebab531fad4f390791e184cd408` (withdrawal) | - |

## Key UTxOs

### Tank Reference Script
- **UTxO**: `354ffe7958d62a8a2bf0b0bd97a06694d59dc49b6d02f1ab40165a3955257168#0`
- **Address**: `addr1w8uhynz89x08gh95758e6dkthtwdlplqzk5an8vj0hzwsesq3894w`
- Contains: Tank reference script (PlutusV3, 6961 bytes)

### Parameters NFT
- **UTxO**: `b79f33b820dd572394cf93e8a4ad1a67ee2d46dbf69ac6ceca33de0d6ff56476#0`
- **NFT**: `f0e403df77b2bfee7c0799ef927b3763165033cbe38bddc802934883.706172616d6574657273` ("parameters")
- **Datum (DatumParameters)**:
  - `min_to_stake`: 30000000000 (30k FLDT)
  - `owner`: `1c471b31ea0b04c652bd8f76b239aea5f57139bdc5a2b28ab1e69175`
  - `address_rewards`: script=`d5940dc2ad9e3f43544fb02a28240f215602603a7519588532b72c53` stake=`5f4dc05250e5a14f74735370134ad5f9506bef2c69e0bca296965f3d`
  - `min_ada`: 1500000 (1.5 ADA)

### Oracle (Charli3 FLDT feed)
- **UTxO**: `7f3bb225b601685e5212935db87b509f7c00fcbb05c36126eb16be1e98aedcc2#0`
- **Policy**: `93794f9b7f3dc632cb889c7aec7d334f016f532e64f16141b6895f5b`
- **Asset name**: `6f7261636c65464c44544333` ("oracleFLDTC3")

### Oracle Contract
- **UTxO**: `fef0fcfc38cace7e8aac85ddc6fad895ca4e7e80a0c01a5836094e387d38f591#0`
- Script hash matches withdrawal credential `1d36e2cb1ad625908e3784eb8bceeebab531fad4f390791e184cd408`

## FLDT Token
- **Policy**: `577f0b1342f8f8f4aed3388b80a8535812950c7a892495c0ecdf0f1e`
- **Asset name (CIP-68)**: `0014df10464c4454`

## Types (on-chain)

### DatumTank
```
DatumTank {
  allowedTokens: List<CardanoToken>,   -- tokens accepted for babel fees
  tankOwner: Address,                   -- who receives token payments
  whitelistedAddresses: List<Address>,  -- addresses allowed to use this tank
  executionTime: Int,                   -- 0 for babel fee, future timestamp for scheduled
  destionationaAddress: Address,        -- destination for scheduled tx
  scheduledAmount: CardanoToken,        -- amount to send in scheduled tx
  reward: CardanoToken,                 -- reward for the batcher node
}
```

### CardanoToken
```
CardanoToken {
  policyId: PolicyId,
  assetName: AssetName,
  amount: Int,       -- fixed price or 0 (use oracle)
  divider: Int,      -- price denominator
  oracle: Option<Asset>,  -- oracle NFT if oracle-based pricing
}
```

### RedeemerTank (constructor ordering)
```
0 = Consume(payingTokenIndex, inputTankIndex, receivers, reference_params_index, whitelist_index)
1 = ConsumeAll(payingTokenIndex, inputTankIndex, receivers, reference_params_index, whitelist_index)
2 = Withdraw
3 = ScheduledTransaction(inputTankIndex, batcher: Address, reference_staking_index, reference_params_index, whitelist_index)
4 = ConsumeOracle(payingTokenIndex, inputTankIndex, receivers, oracleIndex, reference_params_index, whitelist_index)
5 = ConsumeAllOracle(payingTokenIndex, inputTankIndex, receivers, oracleIndex, reference_params_index, whitelist_index)
```

### OracleRedeemer (withdrawal)
```
OracleRedeemer {
  data: OraclePriceFeed,     -- price feed variant
  signatures: List<Signature> -- ed25519 signatures (empty for PriceDataCharlie)
}

OraclePriceFeed variants:
  0 = Aggregated { common, price_in_lovelaces, denominator }
  1 = Pooled { common, token_a_amount, token_b_amount }
  2 = Dedicated { common, price_in_lovelaces, denominator }
  3 = PriceDataCharlie { provider_ref_input_index, common, price_in_lovelaces, price_denominator }
  4 = PriceDataOrcfax { pointer_ref_input_index, provider_ref_input_index, common, price_in_lovelaces, price_denominator }

CommonFeedData { valid_from, valid_to, token: Asset { policyId, assetName } }
```

### StakerRedeemer (mint)
```
RedeemerStaker {
  owner: Address,          -- staker wallet
  signer_bot: Address,     -- bot wallet
  reference_index: Int,    -- index of params ref input
  output_staking: Int,     -- index of staker output
  staking_inputs: Int,     -- number of staking inputs (for burn)
}
```

## Reference Transactions

### ConsumeOracle (Babel Fee with Oracle)
- **Tx**: [`bf7b20eb2c69bded174ea2e560f974973d1cede486bd30fcf712f85a40fbf28e`](https://cexplorer.io/tx/bf7b20eb2c69bded174ea2e560f974973d1cede486bd30fcf712f85a40fbf28e)
- Inputs: 2 (tank + user wallet)
- Outputs: 4 (tank return, payment to owner, user dest, user change)
- Ref inputs: 5 (tank_ref, oracle_feed, oracle_provider, oracle_contract, params)
- Withdrawal: oracle credential with 0 ADA + OracleRedeemer(PriceDataCharlie)
- Required signer: user **payment key** hash
- Validity range: required (for oracle time validation)

### ScheduledTransaction
- **Tx**: [`9899410992740c6166116bb95719fc3b06c3d8cde1714e51c3b3666478f50916`](https://cexplorer.io/tx/9899410992740c6166116bb95719fc3b06c3d8cde1714e51c3b3666478f50916)
- Inputs: 2 (tank + batcher wallet)
- Outputs: 3 (destination, rewards, batcher change)
- Ref inputs: 3 (staker, tank_ref, params)
- Required signer: batcher **staking key** hash
- Validity range: lower bound must be > executionTime

### Staker Mint (Stake FLDT)
- **Tx**: [`727849e845718ebde417c0d15ba0368b6cfcdf08b394d3f5121bd53aac761c94`](https://cexplorer.io/tx/727849e845718ebde417c0d15ba0368b6cfcdf08b394d3f5121bd53aac761c94)
- Input: 1 (user wallet with 30k+ FLDT)
- Outputs: 2 (staker contract: FLDT + NFT, user change)
- Minting: +1 staker NFT (policy=staker_hash, name=bot_stake_key_hash)
- Ref inputs: 1 (params)
- Required signer: user **staking key** hash

### Tank Deposit
- **Tx**: [`ceaa2bcba90105401a8e8ca88c06f3d87ed3e16f7ebcd2bc94e33225c9086272`](https://cexplorer.io/tx/ceaa2bcba90105401a8e8ca88c06f3d87ed3e16f7ebcd2bc94e33225c9086272)
- Simple wallet tx, no script execution
- Output to tank script address (script + user staking credential) with inline datum

## Tank UTxO Statistics
- Total: 808
- Babel fee tanks: 358 (allowedTokens non-empty)
- Scheduled tanks: 445 (executionTime > 0)
- No datum: 8
- Past due scheduled (unexecuted): 407
