# Fluid Aquarium

[FluidTokens](https://fluidtokens.com/) builds a suite of DeFi primitives on Cardano. The **Aquarium** is FluidTokens' infrastructure for two adjacent products: **babel fees** (paying transaction fees in native tokens instead of ADA, via a network of bots that swap tokens against an oracle price) and **scheduled transactions** (transactions queued on-chain today and executed by a third-party network at a specified future time).

Users participate by creating "tanks" — on-chain UTxOs holding ADA, with an inline datum describing what bots are allowed to do with that ADA. Babel-fee bots and Aquarium nodes earn yield by servicing the queue; users earn convenience (paying fees in tokens, automating future transactions) without giving up custody of the funds while they're queued. FLDT, FluidTokens' governance token, is used to register as an Aquarium node operator.

This tx3 covers tank creation (both flavours), user-side reclamation, oracle-driven babel-fee consumption, scheduled-transaction execution, and FLDT staking for node operators.

## Overview

A "tank" is a UTxO sitting at one of the Aquarium validator addresses, holding ADA plus an inline datum that authorises specific consumers. Tank creation is script-free — a plain payment to the validator. The interesting validators run when bots consume tanks:

- **Babel fees:** the bot proves the swap is fair against a Charli3 oracle price via a withdrawal-based oracle validation, then takes ADA from the tank and returns the corresponding token amount to the owner.
- **Scheduled transactions:** an Aquarium node executes the queued transaction once its scheduled time has passed; the validator checks the node's signature against the registered operator set.

FLDT staking is the registration path for node operators; it locks FLDT tokens to mark a key as a valid Aquarium node.

## Transactions

| Transaction | Description |
|---|---|
| `create_babel_tank` | Create a tank for babel-fee payments (ADA deposit + datum) |
| `create_scheduled_tank` | Create a tank for scheduled-transaction execution |
| `withdraw_tank` | Reclaim ADA from an owned tank |
| `consume_oracle` | Batcher consumes part of a tank's ADA, paying the owner in tokens at the oracle rate |
| `execute_scheduled` | Batcher executes a scheduled transaction after its execution time |
| `stake_fldt` | Stake FLDT tokens to become an Aquarium node operator |

## Important considerations

- **Tank creation is script-free.** `create_babel_tank` and `create_scheduled_tank` are plain payments with inline datums — no validators execute.
- **Oracle validation via withdrawal.** `consume_oracle` uses a withdrawal-based oracle (0 ADA withdrawal with a redeemer). The oracle redeemer currently supports Charli3 price feeds.
- **Raw CBOR datums for tank creation.** Tank-creation transactions use raw CBOR for the datum (`tank_datum_cbor`) because the `TankDatum` type contains nested lists, optional types, and on-chain `Address` structures that are too complex to pass as individual parameters.
- **Oracle datum not readable via `datum_is`.** The Charli3 oracle provider datum uses a Map-based CBOR format (`Constr(0, [Constr(2, [Map {...}])])`) that tx3 types cannot model (they require positional constructor fields). Oracle values must still be passed as individual params.
- **Reference inputs ordering.** `consume_oracle` relies on multiple reference inputs (oracle provider, oracle contract, staker, parameters NFT, tank ref script). Their indices in the transaction must match the redeemer fields.
- **Staker redeemer.** `stake_fldt` uses a raw CBOR redeemer (`staker_redeemer_cbor`) due to tx3 limitations on custom types as parameters.
- **Staking-key signer.** `execute_scheduled` and `stake_fldt` need the staker's staking key hash as required signer, but tx3's `signers` block only extracts payment keys — so the caller passes `signer_hash: Bytes` explicitly.
- **Collateral must be pure ADA.** Collateral resolution uses a wallet UTxO directly; if all UTxOs hold native tokens, resolution fails. Use a wallet with at least one pure-ADA UTxO.

## Caller preparation

Several values must be prepared off-chain before invoking transactions.

### `create_babel_tank` / `create_scheduled_tank`

| Parameter | Source |
|---|---|
| `tank_datum_cbor: Bytes` | Full `TankDatum` serialised as raw CBOR by the caller. The type contains nested lists, optional types, and on-chain `Address` structures too complex to pass as individual parameters. |

### `consume_oracle`

| Parameter | Source |
|---|---|
| `oracle_price`, `oracle_denominator`, `oracle_valid_from`, `oracle_valid_to`, `oracle_token_policy`, `oracle_token_name` | Queried from the Charli3 oracle datum on-chain. |
| `input_tank_idx`, `oracle_idx`, `ref_params_idx`, `oracle_provider_idx`, `paying_token_idx` | Reference-input indices that must match the actual ordering of inputs in the built transaction. |
| `payment_ada`, `payment_token_qty`, `tank_return_ada`, `dest_ada` | Computed from the oracle price and the tank contents. |

### `execute_scheduled`

| Parameter | Source |
|---|---|
| `batcher_addr_cbor: Bytes` | Batcher's on-chain `Address` CBOR-encoded externally (tx3 cannot construct Plutus `Address` types as params). |
| `signer_hash: Bytes` | Batcher's **staking key hash** (not payment key). tx3's `signers` block only extracts payment keys. |

### `stake_fldt`

| Parameter | Source |
|---|---|
| `staker_redeemer_cbor: Bytes` | Staker redeemer serialised as raw CBOR due to tx3 limitations on custom types as parameters. |
| `signer_hash: Bytes` | User's **staking key hash** (same reason as above). |

## References

- **Smart contracts:** PlutusV3 (Aiken v1.1.9) — [FluidTokens/ft-cardano-aquarium-sc](https://github.com/FluidTokens/ft-cardano-aquarium-sc)
- **Homepage / app:** [fluidtokens.com](https://fluidtokens.com/)
- **Research notes:** [`investigacion/`](./investigacion/) — oracle datum decoding and reference-input ordering analysis.
