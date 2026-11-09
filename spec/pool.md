# Pool

## Purpose

The pool is Blend v2's isolated lending market: suppliers deposit reserve assets, borrowers draw against collateral, and Dutch auctions settle liquidations, bad debt and accrued interest. v3 binds each pool at deploy to its own backstop, the treasury, the distributor and an optional hook, records a per-reserve `protocol_credit` sold in protocol interest auctions, calls the hook after every batch, and lets an asset's issuer claw back one holder's position.

## Interface

These v2 functions are unchanged: `propose_admin`, `accept_admin`, `cancel_set_reserve`, `set_reserve`, `get_admin`, `get_reserve_list`, `get_positions`, `set_emissions_config`, `get_reserve_emissions`, `get_user_emissions` and `get_auction`.

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `admin: Address, name: String, oracle: Address, bstop_rate: u32, max_positions: u32, min_collateral: i128, min_backstop: i128, backstop_id: Address, treasury: Address, distributor: Address, blnd_id: Address, hook: Option<Address>` | none | admin; called by the factory | Stores the config and the bindings. Rejects a `bstop_rate` above `MAX_BACKSTOP_RATE` and a negative `min_backstop`. |
| `update_pool` | `backstop_take_rate: u32, max_positions: u32, min_collateral: i128` | none | admin | As v2; rejects a backstop take rate above `MAX_BACKSTOP_RATE`. |
| `queue_set_reserve` | `asset: Address, metadata: ReserveConfig` | none | admin | As v2. Rejects a `max_price_age` of zero or a negative `borrow_limit`. |
| `get_config` | none | `PoolConfig` | none | As v2, with `min_backstop`. |
| `get_reserve` | `asset: Address` | `Reserve` | none | As v2, with v3's `ReserveConfig` and `ReserveData` fields. |
| `get_borrow_capacity` | `asset: Address` | `i128` | none | What can still be borrowed of the reserve now, floored at zero. `i128::MAX` on a reserve without a limit. |
| `submit` | `from: Address, spender: Address, to: Address, requests: Vec<Request>` | `Positions` | `spender`, and `from` when different | As v2, plus request type 10, which fills a protocol interest auction; then one hook call. |
| `submit_with_allowance` | `from: Address, spender: Address, to: Address, requests: Vec<Request>` | `Positions` | `spender`, and `from` when different | As `submit`. |
| `flash_loan` | `from: Address, flash_loan: FlashLoan, requests: Vec<Request>` | `Positions` | `from` | As v2; then one hook call. |
| `update_status` | none | `u32` | none | As v2, with the threshold met when the backstop's `total_assets` is at least `min_backstop`, and the queued fraction the backstop's own share balance over its `total_supply`. |
| `set_status` | `pool_status: u32` | none | admin | As v2, with `update_status`'s threshold and queued fraction. |
| `gulp` | `asset: Address` | `i128` | none | As v2; `protocol_credit` is not surplus. On a reserve with `gulp_to_suppliers` set and any supply, the surplus raises `b_rate` instead of `backstop_credit`. |
| `gulp_emissions` | none | `i128` | none | Calls `distributor.gulp_pool_emissions(pool)` and books the amount into the reserve emission indices; no BLND moves. |
| `claim` | `from: Address, reserve_token_ids: Vec<u32>, to: Address` | `i128` | `from` | As v2; transfers BLND from the distributor by `transfer_from` against the pool's allowance. |
| `new_auction` | `auction_type: u32, user: Address, bid: Vec<Address>, lot: Vec<Address>, percent: u32` | `AuctionData` | none | As v2, plus auction type 3, the protocol interest auction. |
| `del_auction` | `auction_type: u32, user: Address` | none | none | As v2. |
| `bad_debt` | `user: Address` | none | none | As v2; the backstop's debt is defaulted to suppliers only when the backstop's `total_assets` is zero, or less than the fraction of `min_backstop` fixed in pool code. |
| `clawback` | `asset: Address, from: Address, amount: i128, repay: Map<Address, i128>` | none | the current administrator of the asset's Stellar Asset Contract | Repays the listed debts of `from` from the administrator, removes `amount` of `from`'s supplied position and burns `amount` from the pool's custody through the SAC. |
| `reconcile` | `asset: Address` | `i128` | none | When the pool holds less of `asset` than it owes, writes the shortfall off against `protocol_credit`, then suppliers through `b_rate`, then `backstop_credit`, and, when the shortfall is at least `freeze_threshold` of the reserve's total supply, sets the pool's status to admin frozen. Deletes an open interest auction whose lot it cut. Returns the shortfall; without one it returns zero and changes nothing. |
| `get_backstop` | none | `Address` | none | The backstop. |
| `get_treasury` | none | `Address` | none | The treasury. |
| `get_distributor` | none | `Address` | none | The distributor. |
| `get_hook` | none | `Option<Address>` | none | The hook, or `None` for a pool without one. |

**Interest accrual.** When a reserve accrues, the pool reads `treasury.get_rate()`, caps it at `MAX_PROTOCOL_RATE`, and splits the accrued interest in the reserve's token: `backstop_credit` at the backstop take rate, `protocol_credit` at the protocol rate, both rounded down, and the remainder to suppliers through `b_rate`. The pool keeps no copy of the rate. A backstop take rate of zero does not change the protocol share.

**Backstop reads and calls.** The pool reads the backstop through the vault standard: `query_asset` for the bid token of a backstop interest auction and the lot token of a bad-debt auction; `total_assets` for the activation threshold, the bad-debt lot cap and the default rule; the backstop's own share balance over `total_supply` for the queued fraction that moves status. `asset_price` prices backstop tokens for both auctions: the pool rescales it from the decimals it reports to the oracle's before sizing the bid or the lot. The pool's oracle must quote in USD. A backstop interest auction fill transfers the scaled bid from the filler to the backstop under the filler's authorization, with no backstop call and no allowance; a bad-debt auction fill calls `draw`. The pool calls no other backstop function.

**Protocol interest auction.** `new_auction(3, treasury, bid, lot, 100)` creates one. The lot is each listed reserve's `protocol_credit` after accrual, skipping reserves with none. It must be worth at least the minimum a backstop interest auction needs, at the pool oracle's prices. The bid is USDC, `treasury.get_token()`, sized at 1.2 times the lot's value, with one unit of the oracle's base worth one USDC, rescaled from the oracle's decimals to 7. One runs at a time. Bid and lot scale along v2's auction curve. Request type 10 fills it: the pool calls `treasury.donate(filler, scaled bid)` under the filler's authorization, subtracts each delivered lot amount from its reserve's `protocol_credit`, and transfers the lot to the filler.

**Interest auction fills.** Backstop and protocol interest auctions fill only in full. A fill below 100% fails with `BadRequest`.

**Borrow limit.** On a reserve with a `borrow_limit` above zero, `net` is the reserve's borrows, a flash loan included, minus its repayments over one `submit` or `flash_loan`, floored at zero. At the end of the batch the pool sets `used` to `max(0, used − ceil(borrow_limit × elapsed / BORROW_WINDOW)) + net`, where `elapsed` is the time since `last`. It fails with `BorrowLimitExceeded` if `net` is above zero and `used` exceeds `borrow_limit`, and otherwise stores `used` with the current time. A repayment frees capacity only against borrows in the same batch. An auction fill is not a borrow. A missing entry means nothing is in use. A change to `borrow_limit` keeps `used` and `last`. `BORROW_WINDOW` is a constant, five hours.

**Hook call.** After every request in a batch has been applied, the health check has passed and the tokens have moved, the pool calls `hook.on_submit(from, spender, to, requests, reserves, positions)` once, with the batch as submitted, every reserve it touched in its final state and the final positions of `from`. Every batch is hooked, whatever its request types; no other function calls the hook. A panic reverts the whole transaction. Without a hook no call is made (spec/hooks.md).

### Clawback

**Authorization.** The pool calls `admin()` on `asset` and requires that address's authorization. The administrator authorizes the pool call together with the SAC `clawback(pool, amount)` call the pool makes inside it. A token without an `admin()` function fails the call. A balance that was not created with the clawback flag fails inside the SAC; the pool does not check the flag itself. The pool admin, the hook, the treasury owner and the factory have no path to this entry point.

**Amount and rounding.** `amount` is in the reserve's underlying token and must be positive. The pool loads the reserve, which accrues interest to the current ledger, and converts `amount` to b_tokens rounding up, as a withdraw does. `from` must hold at least that many b_tokens across ordinary supply and collateral. A smaller position fails the call; the pool does not clamp to what `from` holds. The b_tokens burned are worth at least `amount` at the reserve's `b_rate`; the difference, under one b_token, stays with the pool as a withdraw's rounding does.

**Repayment.** `repay` maps a reserve asset to the amount of it the administrator repays on `from`'s debt before the burn; an empty map repays nothing. Each entry is applied as a `Repay` request with the administrator as spender: it burns `from`'s d_tokens, and any amount above the debt is returned to the administrator. The administrator authorizes these transfers together with the call.

**Burn order.** Ordinary supply is burned first. What remains is burned from collateral. Before the burn the pool settles `from`'s accrued emissions on the reserve's supply token, as it does for any removal of b_tokens. A position that reaches zero is removed from `from`'s position set. The reserve's `b_supply` falls by the b_tokens burned. After the burn the reserve's utilization must be below 100%, as after a withdraw.

**Custody burn and its check.** The pool reads its own balance of `asset`, calls the SAC's `clawback(pool, amount)`, reads the balance again and requires that it fell by exactly `amount`. No tokens are transferred; the SAC burns them. The pool must hold `amount` of the asset at call time. Tokens borrowed out of the pool are not in its custody and cannot be recalled through it.

**Auctions and health.** After the burn the pool checks `from`'s health as after a collateral withdrawal; a clawback that would leave `from` below the minimum health factor fails with `InvalidHf`. If `from` has an open user liquidation auction the call fails with `AuctionInProgress`. The clawback changes no auction. Backstop interest, protocol interest and bad-debt auctions are untouched: their lots and bids are `backstop_credit`, `protocol_credit` and the backstop's positions, which the clawback does not change.

**Status and hook.** The call is allowed in every pool status and on a disabled reserve, as a withdraw is. It is not a `submit`: the hook is not called and cannot block or trigger it.

**What it cannot do.** It cannot recall borrowed-out tokens, erase any debt, change another holder's position, touch `backstop_credit`, `protocol_credit` or the backstop's deposits, or act on a token that has no SAC administrator, such as the v1.1 Comet LP token. It cannot freeze a balance. An issuer action against the pool's own balance: see `reconcile` and Open points.

## Types

| Type | Fields | Content |
|---|---|---|
| `PoolConfig` | v2's fields plus `min_backstop: i128` | v2's pool config; `min_backstop` is the activation threshold, in backstop tokens |
| `ReserveConfig` | v2's fields plus `max_price_age: u64`, `gulp_to_suppliers: bool`, `freeze_threshold: u32` and `borrow_limit: i128` | v2's reserve config. `max_price_age` is how old a price for the reserve may be, in seconds. `gulp_to_suppliers` sends `gulp`'s surplus to suppliers instead of `backstop_credit`. `freeze_threshold` is the shortfall, as a share of the reserve's total supply in 7 decimals, from which `reconcile` freezes the pool, and 0 never freezes. `borrow_limit` is the most net borrowing the reserve allows at once, in the asset's units. Capacity refills at `borrow_limit` per `BORROW_WINDOW`. 0 has no limit. |
| `BorrowUse` | `used: i128, last: u64` | the part of a reserve's borrow limit in use and the time it was last updated |
| `ReserveData` | v2's fields plus `protocol_credit: i128` | v2's reserve data; `protocol_credit`, next to `backstop_credit`, is the protocol's share of accrued interest, in the reserve's token |
| `RequestType` | v2's variants plus `FillProtocolInterestAuction = 10` | the request types `submit` accepts |
| `AuctionType` | v2's variants plus `ProtocolInterestAuction = 3` | the auction types |

## Storage

| Key | Type | Content |
|---|---|---|
| `Admin`, `PropAdmin`, `Name`, `Config`, `ResList`, `Backstop`, `BLNDTkn`, `PoolEmis` | as v2 | v2's entries; `Config` carries `min_backstop` |
| `Treasury` | instance, `Address` | the treasury |
| `Distributor` | instance, `Address` | the distributor |
| `Hook` | instance, `Option<Address>` | the hook, if any |
| `BorrowUse(asset)` | temporary, `BorrowUse` | the reserve's borrow limit in use. Its TTL is at least `BORROW_WINDOW` in ledgers at the shortest ledger time. |
| v2's persistent keys, emissions included | persistent, as v2 | `ReserveData` carries `protocol_credit` |

No protocol rate is stored.

## Events

| Event | Topics | Data |
|---|---|---|
| every v2 event | as v2 | as v2 |
| `clawback` | `["clawback", asset, from]` | `[amount, supply_b_tokens, collateral_b_tokens]` |
| `reconcile` | `["reconcile", asset]` | `[shortfall, protocol_credit_loss, supplier_loss, backstop_credit_loss]` |

`new_auction`, `fill_auction` and `delete_auction` carry auction type 3 for a protocol interest auction. `supply_b_tokens` and `collateral_b_tokens` are the b_tokens burned from ordinary supply and from collateral. The SAC emits its own `clawback` event with the pool as `from`. Each repayment in a `clawback` emits v2's `repay` event. An auction `reconcile` deletes emits `delete_auction`. A freeze by `reconcile` emits v2's `set_status` event. A hook call emits nothing from the pool.

## Errors

| Error | Code | When |
|---|---|---|
| `InvalidPoolConfigArgs` | 1201 | as v2, plus a negative `min_backstop` or a backstop take rate above the cap |
| `InvalidReserveMetadata` | 1202 | as v2, plus a `max_price_age` of zero or a negative `borrow_limit` |
| `InvalidPrice` | 1210 | as v2, with the reserve's `max_price_age` in place of 24 hours |
| the backstop interest auction's codes | as v2 | the same conditions in a protocol interest auction |
| `BadRequest` | 1200 | as v2, plus a `clawback` whose `amount` is not positive or whose `from` is the pool, and an interest auction fill below 100% |
| `BalanceError` | 10 | as v2, plus a `clawback` whose `from` holds fewer b_tokens than `amount` converts to, or after which the pool's balance did not fall by exactly `amount` |
| `InvalidHf` | 1205 | as v2, plus a `clawback` that would leave `from` below the minimum health factor |
| `MinCollateralNotMet` | 1224 | as v2, plus a `clawback` that would leave `from` with debt and collateral below `min_collateral` |
| `InvalidUtilRate` | 1207 | as v2, plus a `clawback` that leaves the reserve's utilization at or above 100% |
| `AuctionInProgress` | 1212 | as v2, plus a `clawback` while `from` has an open user liquidation auction |
| `BorrowLimitExceeded` | 1225 | a borrow that takes a reserve's `used` above its `borrow_limit` |

`PoolError` keeps v2's codes. A failing `treasury.donate` propagates unchanged. In `clawback` the SAC rejects a missing administrator signature, a short pool balance or a non-clawbackable balance. A token without `admin()` is a host error. A rejecting hook's own code propagates unchanged.

## Invariants

- The backstop, treasury, distributor, hook and `min_backstop` are written once by the constructor and never changed. The pool has no upgrade.
- Per accrual, the protocol share is at most `accrued × MAX_PROTOCOL_RATE` and the backstop share at most `accrued × MAX_BACKSTOP_RATE`; the two constants sum to less than the whole, so the suppliers' remainder is never negative.
- The rate the treasury reports at accrual applies to the whole interval since the reserve's last accrual.
- `protocol_credit` is never negative, rises only through accrual and falls only through a protocol interest auction fill or `reconcile`; it is excluded from the supplier total and from utilization, as `backstop_credit` is. The auction never sells supplier funds, and its bid goes to the treasury only.
- The hook cannot change amounts, accounting or transfers; a panic leaves no trace.
- The activation threshold is `total_assets ≥ min_backstop`, with no price. `min_backstop` 0 always meets it, and the backstop's holdings are still auctioned before any debt is defaulted.
- Backstop tokens in bad-debt and backstop interest auctions are valued at the backstop's `asset_price`, rescaled to the oracle's decimals, as v2 valued them at `token_spot_price`. A backstop interest auction fill moves the bid by one transfer, from the filler to the backstop.
- `clawback` burns supply worth at least `amount` and exactly `amount` of custody, never leaves `from` below the minimum health factor, changes `from`'s liabilities only by the administrator's repayment and never calls the hook.
- `reconcile` changes nothing unless the pool holds less of the asset than it owes, and never touches the backstop's deposits.

## Differences from v2

Cited files are in `blend-contracts-v2`.

- Bindings: `__constructor` (`pool/src/contract.rs`) took `backstop_id` and `blnd_id` from the factory; v3 adds `min_backstop`, `treasury`, `distributor` and `hook` (`pool/src/storage.rs`).
- Accrual: `Reserve::accrue` (`pool/src/pool/reserve.rs`) took one share, at `bstop_rate`; v3 takes a second, at the treasury's rate, into `protocol_credit`.
- Rate caps: `require_valid_pool_config` (`pool/src/pool/config.rs`) accepted any `bstop_rate` below one. v3 caps it at `MAX_BACKSTOP_RATE`, 50%, and the protocol rate at `MAX_PROTOCOL_RATE`, 5%, constants in `pool/src/constants.rs`.
- Price age: `load_price` (`pool/src/pool/pool.rs`) rejected a price older than 24 hours; v3 uses the reserve's `max_price_age`.
- Gulp: `execute_gulp` (`pool/src/pool/gulp.rs`) credited every surplus to `backstop_credit`; v3 raises `b_rate` instead on a reserve with `gulp_to_suppliers` set.
- Auctions: `AuctionType` (`pool/src/auctions/auction.rs`) had three types and `RequestType` (`pool/src/pool/actions.rs`) ten; v3 adds one of each, built on `pool/src/auctions/backstop_interest_auction.rs`.
- Fills: `fill` (`pool/src/auctions/auction.rs`) accepts any percent from 1 to 100 for every auction type. v3 accepts only 100 for interest auctions.
- Backstop calls: `backstop_interest_auction.rs` and `bad_debt_auction.rs` checked the bid or lot token against `backstop.backstop_token()` and sized with `backstop.pool_data().token_spot_price` at 7 decimals; v3 uses `query_asset()` and `asset_price()`, rescaled. The interest auction's fill called `backstop.donate` under the filler's allowance; v3 transfers the bid to the backstop. `status.rs` read `pool_data().q4w_pct`; v3 divides the backstop's share balance by `total_supply()`.
- Threshold: `calc_pool_backstop_threshold` (`pool/src/pool/status.rs`) computed a product constant from the BLND and USDC behind the backstop's LP tokens; v3 compares `backstop.total_assets()` with `min_backstop`, and the default rule in `pool/src/pool/bad_debt.rs` follows.
- Emissions: `pool/src/emissions/manager.rs` called `backstop.gulp_emissions(pool)` for the allocation and `distributor.rs` drew the BLND from the backstop's allowance; v3 calls `distributor.gulp_pool_emissions(pool)` and draws from the distributor's. Backstop depositors claim from the distributor (spec/distributor.md).
- Hook: new, at the end of `execute_submit` and `execute_submit_with_flash_loan` (`pool/src/pool/submit.rs`). `pool/src/dependencies/` gains treasury, distributor and hook clients.
- Clawback: v2 has no issuer entry point. A clawback executed directly against the pool's balance leaves custody short of the pool's records with no accounting response: `gulp` in `pool/src/pool/gulp.rs` credits a surplus to `backstop_credit` and returns zero on a deficit. A deauthorization of the pool's balance stops the pool's transfers of that asset, with no accounting response. v3 writes the first off with `reconcile` and leaves the second; the targeted clawback is added beside them. Against a v2 withdraw (`apply_withdraw` in `pool/src/pool/actions.rs`), the clawback keeps the conversion (`to_b_token_up` in `pool/src/pool/reserve.rs`), the emission settlement (`remove_supply` and `remove_collateral` in `pool/src/pool/user.rs`), the utilization rule (`require_utilization_below_100` in `pool/src/pool/reserve.rs`) and the health check (`validate_submit` in `pool/src/pool/submit.rs`). It differs in five ways: the SAC administrator authorizes it instead of the holder; it does not clamp to the position; the SAC burns the tokens instead of the pool transferring them; one call spans ordinary supply and collateral, in that order; and it is not a `submit`, so no hook runs. An open user liquidation auction on `from` blocks it as it blocks a `submit` (`validate_submit` in `pool/src/pool/submit.rs`). Repayments follow `apply_repay` in `pool/src/pool/actions.rs`, with the administrator as spender.
- Shortfall: `reconcile` is new; a shortfall of at least the reserve's `freeze_threshold` sets status 4, admin frozen, which `update_status` (`pool/src/pool/status.rs`) cannot lift.
- Borrow limit: new. v2 limits a reserve's borrowing only by `max_util` (`require_utilization_below_max` in `pool/src/pool/reserve.rs`), not by how fast debt grows.

## Open points

- Issuer action against the pool itself (README open question 3): a freeze of the pool's own balance is not handled in code; whether to add quarantine. Listing an asset whose administrator holds those powers is the pool admin's decision and needs the issuer's agreement to use the pool's entry point (README §3.6).
- Yield paid in kind (README open question 4): the surplus raises `b_rate` in one step; whether to spread it over time.
