# Fluid Aquarium

tx3 protocol for [FluidTokens Aquarium](https://github.com/FluidTokens/ft-cardano-aquarium-sc) — babel fees and scheduled transactions on Cardano.

> **Status: WIP** — This tx3 off-chain implementation is under testing and may not cover all edge cases yet.

## Overview

Fluid Aquarium allows users to create "tanks" (UTxOs with ADA) that can be consumed by:

- **Babel fee bots:** Pay a user's transaction fees in exchange for native tokens at an oracle-determined rate
- **Aquarium nodes:** Execute scheduled transactions at a specified future time

Users can also stake FLDT tokens to register as Aquarium node operators.

## Transactions

| Transaction | Description |
|---|---|
| `create_babel_tank` | Create a tank for babel fee payments (ADA deposit + datum) |
| `create_scheduled_tank` | Create a tank for scheduled transaction execution |
| `withdraw_tank` | Reclaim ADA from an owned tank |
| `consume_oracle` | Batcher consumes part of a tank's ADA, paying the owner in tokens at oracle rate |
| `execute_scheduled` | Batcher executes a scheduled transaction after its execution time |
| `stake_fldt` | Stake FLDT tokens to become an Aquarium node operator |

## Important Considerations

- **Tank creation is script-free:** `create_babel_tank` and `create_scheduled_tank` are plain payments with inline datums — no validators execute.
- **Oracle validation:** `consume_oracle` uses a withdrawal-based oracle (0 ADA withdrawal with redeemer). The oracle redeemer currently supports Charli3 price feeds.
- **Withdrawal redeemer:** Fixed ([tx3#317](https://github.com/tx3-lang/tx3/pull/317)). The reward redeemer is now correctly compiled into the witness set.
- **Raw CBOR datums:** Tank creation transactions use raw CBOR for the datum (`tank_datum_cbor`) because the `TankDatum` type contains nested lists and optional types that are complex to pass as parameters.
- **Oracle datum not readable via `datum_is`:** The Charli3 oracle provider datum uses a Map-based CBOR format (`Constr(0, [Constr(2, [Map {...}])])`) which tx3 types cannot model (they require positional constructor fields). Oracle values must still be passed as individual params.
- **Reference inputs ordering:** `consume_oracle` relies on multiple reference inputs (oracle provider, oracle contract, staker, parameters NFT, tank ref script) — their indices in the transaction must match the redeemer fields.
- **Staker redeemer:** `stake_fldt` uses a raw CBOR redeemer (`staker_redeemer_cbor`) due to tx3 limitations on custom types as parameters.

## Caller Preparation

Several values must be prepared off-chain before invoking transactions:

### `create_babel_tank` / `create_scheduled_tank`

- `tank_datum_cbor: Bytes` — The full tank datum must be serialized as raw CBOR by the caller. The `TankDatum` type contains nested lists, optional types, and on-chain `Address` structures too complex to pass as individual parameters.

### `consume_oracle`

This transaction requires extensive off-chain data:
- **Oracle price data:** `oracle_price`, `oracle_denominator`, `oracle_valid_from`, `oracle_valid_to`, `oracle_token_policy`, `oracle_token_name` — queried from the Charli3 oracle datum on-chain.
- **Reference input indices:** `input_tank_idx`, `oracle_idx`, `ref_params_idx`, `oracle_provider_idx`, `paying_token_idx` — must match the actual ordering of inputs in the built transaction.
- **Payment calculation:** `payment_ada`, `payment_token_qty`, `tank_return_ada`, `dest_ada` — computed from the oracle price and tank contents.

### `execute_scheduled`

- `batcher_addr_cbor: Bytes` — The batcher's on-chain `Address` must be CBOR-encoded externally (tx3 cannot construct Plutus `Address` types as params).
- `signer_hash: Bytes` — The batcher's **staking key hash** (not payment key). tx3's `signers` block only extracts payment keys.

### `stake_fldt`

- `staker_redeemer_cbor: Bytes` — The staker redeemer must be serialized as raw CBOR due to tx3 limitations on custom types as parameters.
- `signer_hash: Bytes` — The user's **staking key hash** (same reason as above).

## tx3 Limitations

Several tx3 language limitations affect this protocol. Three were resolved with unreleased fixes: field name shadowing ([tx3#316](https://github.com/tx3-lang/tx3/pull/316)), withdrawal redeemer generation ([tx3#317](https://github.com/tx3-lang/tx3/pull/317)), and typed reference datum access ([tx3#318](https://github.com/tx3-lang/tx3/pull/318)). Remaining limitations include custom types as params, list values as invoke args, and staking key extraction. For the full list, see [investigacion/tx3-limitations-aquarium.md](investigacion/tx3-limitations-aquarium.md).

## Smart Contracts

- PlutusV3 (Aiken v1.1.9)
- Source: [FluidTokens/ft-cardano-aquarium-sc](https://github.com/FluidTokens/ft-cardano-aquarium-sc)
