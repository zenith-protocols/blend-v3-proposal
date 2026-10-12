# Backstop

## Purpose

A backstop holds one pool's first-loss capital in one token: depositors receive shares, the pool draws on the tokens to cover bad debt, and backstop interest auctions pay the backstop in the same token. The backstop follows the ERC-4626 standard for tokenized vaults, with ERC-7540's asynchronous redemption in front of its exits; cancellation follows ERC-7887. v3 reduces Blend v2's backstop to a single pool, adopts OpenZeppelin's vault standard for shares, deposits and exits, keeps v2's withdrawal queue in front of the exits and reports every depositor's share balance change to the distributor, which keeps the depositors' emissions. The backstop holds no BLND and no emission state.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `pool: Address, backstop_token: Address, distributor: Address, decimals_offset: u32, name: String, symbol: String` | none | the deployer: the factory on the standard route | Binds the pool and the distributor, sets the vault asset, the virtual decimals offset and the share metadata, all immutable. The standard backstop reads USDC from the Comet pool. |
| `deposit` | `assets: i128, receiver: Address, from: Address, operator: Address` | `i128`, shares minted | `operator` | Takes `assets` backstop tokens from `from` and mints `receiver` the shares they are worth, rounding down. Refuses a deposit that mints no shares. Reports `receiver`. |
| `mint` | `shares: i128, receiver: Address, from: Address, operator: Address` | `i128`, backstop tokens taken | `operator` | The same for an exact number of shares: takes the backstop tokens `shares` cost from `from`, rounding up, and mints `shares` to `receiver`. Reports `receiver`. |
| `request_redeem` | `shares: i128, owner: Address, operator: Address` | `RedeemRequest` | `operator` | Moves `shares` of `owner`'s into escrow at the backstop's own address and records an entry that unlocks 17 days later. Open entries per owner are capped as in v2. Reports `owner`. |
| `cancel_redeem_request` | `shares: i128, owner: Address, operator: Address` | none | `operator` | Returns `shares` escrowed shares to `owner`, newest entries first. Reports `owner`. |
| `redeem` | `shares: i128, receiver: Address, owner: Address, operator: Address` | `i128`, backstop tokens paid | `operator` | Burns `shares` of `owner`'s unlocked escrowed shares, oldest entries first, and pays `receiver` the backstop tokens they are worth, rounding down. A partly consumed entry keeps its remainder, still unlocked. Fails while the pool records liabilities for the backstop, above `max_redeem(owner)`, or for a zero payout. |
| `withdraw` | `assets: i128, receiver: Address, owner: Address, operator: Address` | `i128`, shares burned | `operator` | The same for an exact amount of backstop tokens: burns the unlocked escrowed shares `assets` cost, rounding up, oldest entries first, and pays `assets` to `receiver`. Fails above `max_withdraw(owner)`. |
| `max_redeem` | `owner: Address` | `i128` | none | `owner`'s unlocked escrowed shares. |
| `max_withdraw` | `owner: Address` | `i128` | none | The backstop tokens `max_redeem(owner)` is worth. |
| `pending_redeem_request` | `owner: Address` | `i128` | none | `owner`'s escrowed shares still locked. |
| `claimable_redeem_request` | `owner: Address` | `i128` | none | `owner`'s unlocked escrowed shares; equals `max_redeem(owner)`. |
| `unqueued_assets` | none | `i128` | none | `convert_to_assets` of `total_supply` less the backstop's own share balance. |
| `asset_price` | none | `AssetPrice` | none | The price of one backstop token in USD, in `decimals`. |
| `pool` | none | `Address` | none | The bound pool. |
| `distributor` | none | `Address` | none | The distributor. |
| `draw` | `amount: i128, to: Address` | none | the pool | Sends `amount` backstop tokens to `to`; fails if the balance is short. |
| Vault standard | ERC-4626: `query_asset`, `total_assets`, `max_deposit`, `max_mint`, `preview_deposit`, `preview_mint`, `preview_withdraw`, `preview_redeem`, `convert_to_shares`, `convert_to_assets` | as ERC-4626 | none | OpenZeppelin's implementation. |
| Share token | SEP-41 plus `total_supply`: `balance`, `transfer`, `transfer_from`, `burn`, `burn_from`, `approve`, `allowance`, `decimals`, `name`, `symbol` | as SEP-41 | the holder or an approved spender | OpenZeppelin's implementation. `transfer` and `transfer_from` report each party; `burn` and `burn_from` report the holder. A transfer to the backstop's own address is refused. |

A `from` other than the operator provides the deposit through the operator's allowance on the backstop token; an `owner` other than the operator has a redemption requested or cancelled, or shares redeemed or withdrawn, through the operator's allowance on the share token, as the standard spends it.

The standard backstop derives `asset_price` from the Blend v2.1 BLND–USDC Comet pool as v2's `token_spot_price` did: USDC per LP share, scaled by the inverse of USDC's weight in the pool, without fees or slippage, reported at 7 decimals. A custom backstop reports it from an oracle or a pricing method of its own.

The backstop's one emission duty is the report: `distributor.report(pool, user, shares)`, once per reported account, with its own pool and the account's share balance after the change. Nothing outside the backstop can change a share balance, so no other function reports. The backstop reports whether or not its pool is in the zone. The backstop keeps no index, no schedule and no accrual, and never holds BLND. A custom backstop implements `query_asset`, `total_assets`, `total_supply`, `balance` of itself, `draw`, `asset_price` and `pool`, with the pool's address in its constructor; the rest is the pool admin's choice. It reports nothing.

## Types

| Type | Fields | Content |
|---|---|---|
| `RedeemRequest` | `shares: i128, claimable_at: u64` | shares in escrow and the time they become claimable |
| `AssetPrice` | `price: i128, decimals: u32` | the price of one backstop token in USD, in `decimals` |

## Storage

| Key | Type | Content |
|---|---|---|
| `Pool` | instance, `Address` | the bound pool |
| `Distributor` | instance, `Address` | the distributor |
| vault asset, virtual decimals offset, share metadata, total supply | instance, OpenZeppelin entries | |
| `RedeemRequests(owner)` | persistent, `Vec<RedeemRequest>` | `owner`'s open requests, oldest first |
| share balances | persistent, OpenZeppelin entries | |
| allowances | temporary, OpenZeppelin entries | |

No emission data is stored. No token count and no request total are stored: the shares in escrow are the backstop's own share balance.

## Events

| Event | Topics | Data |
|---|---|---|
| `deposit`, `withdraw`, `transfer`, `approve`, `burn` | as OpenZeppelin | as OpenZeppelin |
| `request_redeem` | `["request_redeem", owner]` | `[shares, claimable_at]` |
| `cancel_redeem_request` | `["cancel_redeem_request", owner]` | `shares` |
| `draw` | `["draw"]` | `[to, amount]` |

The queue events are v2's, renamed with their functions. A donation is the backstop token's own `transfer` event; the backstop emits nothing for it. The backstop has no emission event.

## Errors

| Error | Code | When |
|---|---|---|
| `VaultInvalidAssetsAmount`, `VaultInvalidSharesAmount` | 403, 404 | a negative amount in one of the standard's functions |
| `VaultExceededMaxWithdraw`, `VaultExceededMaxRedeem` | 407, 408 | an exit beyond the owner's unlocked escrow |
| `MathOverflow` | 410 | an overflow in the standard's conversions |
| OpenZeppelin token codes | as OpenZeppelin | the share token and its allowances |
| `NegativeAmountError` | 8 | a negative amount in `request_redeem`, `cancel_redeem_request` or `draw` |
| `BalanceError` | 10 | a `request_redeem` above the owner's share balance, or a `cancel_redeem_request` above the owner's escrowed shares |
| `BadRequest` | 1000 | a deposit or mint whose `from` or `receiver` is the pool or the backstop itself, or a share transfer to the backstop itself |
| `InsufficientFunds` | 1003 | a `draw` above the backstop's token balance |
| `InvalidShareMintAmount` | 1005 | a deposit that mints no shares |
| `InvalidTokenWithdrawAmount` | 1006 | a redeem that pays nothing |
| `TooManyRedeemRequests` | 1007 | a request beyond the cap on open requests |
| `BadDebtExists` | 1011 | a `redeem` or `withdraw` while the pool records liabilities for the backstop |

The standard's codes pass through on its functions; v2's codes cover the same conditions elsewhere. `NotExpired` (1001) goes: a locked entry is outside `max_redeem`, so an exit that reaches it fails with the standard's code. The reward-zone codes 1002 and 1009 move to the distributor; `NotPool` (1004) goes with the `is_pool` check; `MaxBackfillEmissions` (1010) goes with the backfill. No emission error remains: a failure of `report` fails the backstop call.

## Invariants

- One backstop serves one pool, fixed at construction. Only the pool can `draw`; the pool and the backstop itself are never `from` or `receiver` of a deposit.
- Shares reach the backstop's own address only through `request_redeem` and leave it only through `cancel_redeem_request`, which returns them to the owner, and `redeem` and `withdraw`, which burn them there.
- An entry unlocks 17 days after the request. Cancellation consumes the newest entries; redeem and withdraw the oldest, unlocked ones only, so `max_redeem` and `max_withdraw` bound every exit and a share never leaves escrow for tokens before its entry unlocks.
- No `redeem` or `withdraw` while the pool holds liabilities against the backstop.
- Every change to a depositor's share balance is reported in the same call, with the balance after it.
- The backstop holds no BLND and no emission state. It never pays, schedules or settles emissions; the report is its only emission call.

## Differences from v2

Compared with the audited v2 code in `blend-contracts-v2`:

- Scope. v2 served every pool from one contract with per-pool balances (`PoolBalance` in `backstop/src/backstop/pool.rs`) and verified pools through `pool_factory.is_pool`. v3 binds one pool at construction and drops both.
- Shares. v2 computed shares in `PoolBalance::convert_to_shares` and held them in a non-transferable `UserBalance` record. v3 uses OpenZeppelin's vault with a virtual offset and issues shares as a SEP-41 token.
- Names. v2's `deposit(from, pool_address, amount)` and `withdraw(from, pool_address, amount)` (`backstop/src/contract.rs`) become the standard's `deposit(assets, receiver, from, operator)` and `redeem(shares, receiver, owner, operator)`, with `mint` and `withdraw` beside them and the standard's `max_`, `preview_` and `convert_` reads. `queue_withdrawal` and `dequeue_withdrawal` become ERC-7540's `request_redeem` and ERC-7887's `cancel_redeem_request`, with ERC-7540's `pending_redeem_request` and `claimable_redeem_request` views beside them. `backstop_token` becomes the standard's `query_asset`.
- Queue. v2 counted queued shares in `UserBalance.q4w` and `PoolBalance.q4w` (`backstop/src/backstop/user.rs`, `pool.rs`); v3 escrows them at the backstop's own address and bounds every exit by `max_redeem` and `max_withdraw`. Lock time and entry limit are v2's constants (`backstop/src/constants.rs`).
- Donate. v2's `execute_donate` (`backstop/src/backstop/fund_management.rs`) pulled the tokens by allowance under the donor's and the pool's authorization and credited the pool's balance. v3 has no donate: a transfer of the backstop token to the backstop is the donation, and the pool's backstop interest auction fill transfers the bid directly.
- Constructor. v2's took the emitter, the BLND and USDC tokens, the factory and a drop list; v3's takes the pool, the distributor, the decimals offset and the share metadata instead. The drop list moves to the distributor; the backfill logic goes.
- `pool_data`. Removed. The pool reads `total_assets`, `total_supply` and the backstop's own share balance in place of `tokens`, `shares` and `q4w_pct`, and the distributor reads `unqueued_assets` and `total_assets`; the `blnd` and `usdc` fields are dropped; `token_spot_price` becomes `asset_price`, computed as in `load_pool_backstop_data` and reported with its decimals. `is_pool_above_threshold` leaves the backstop: activation is the pool's `min_backstop`, admission to the reward zone is the distributor's.
- Emissions. All of `backstop/src/emissions/` moves to the distributor: `distribute`, `add_reward`, `remove_reward` and `gulp_emissions` from `manager.rs`, the per-depositor index with `BackstopEmissionData` and `UserEmissionData` from `distributor.rs`, and `claim` from `claim.rs`. So does `drop` (`backstop/src/contract.rs`). v2's backstop settled a depositor's index before every balance change and `execute_claim` swapped the BLND into LP tokens and redeposited them; v3's backstop settles nothing, holds no BLND and has no `claim` and no `gulp_emissions`. The pool calls the distributor's `gulp_pool_emissions` where it called `backstop.gulp_emissions`; depositors call the distributor's `claim`, which pays BLND.
- Reports. v2's manager read `PoolBalance` and `UserBalance` from its own storage; v3's backstop reports each changed depositor's shares to the distributor, which reads `unqueued_assets` and `total_assets` from the backstop.
