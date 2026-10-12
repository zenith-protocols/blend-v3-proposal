# Distributor

## Purpose

The distributor is the emitter's recipient: it receives the BLND the emitter mints and splits it across the reward zone under v2's rules, applied to separate backstops instead of one. Only a standard deployment of the factory can enter the zone; a custom backstop earns nothing, even one holding the same LP token. Backstops report their depositors' shares to it under their pool's address. The distributor keeps, per pool, the reported balances, each depositor's accrual and the pool's own; depositors claim from it, pools pull their share from it as an allowance. It is the only holder of BLND between the emitter and the claimants.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `emitter: Address, blnd_token: Address, usdc_token: Address, lp_token: Address, factory: Address, drop_list: Vec<(Address, i128)>` | none | the deployer | Stores the dependencies and the drop list; `lp_token` is the Blend v2.1 BLND–USDC Comet pool, whose LP token is the standard backstop asset. No setters. |
| `report` | `pool: Address, user: Address, shares: i128` | none | the address `factory.get_backstop(pool)` returns | Calls `factory.get_backstop(pool)`, which panics unless `pool` is a standard deployment, and requires the returned address's authorization. Updates the pool's index and `user`'s accrual at the stored shares, as v2's `update_emissions`, then stores `shares` as `user`'s and moves the pool's share total by the difference. Fails for a negative amount. |
| `claim` | `from: Address, pools: Vec<Address>, to: Address` | `i128`, BLND paid | `from` | Updates each listed pool's index and `from`'s accrual as a report does, takes what `from` has accrued in each and transfers the sum to `to` in one BLND transfer. Fails on an empty list, a pool listed twice, a pool without emission data and a zero sum. |
| `gulp_pool_emissions` | `pool: Address` | `i128`, BLND approved | `pool` | Splits the BLND allotted to `pool` since its last gulp as v2's `gulp_emissions`: raises `pool`'s allowance on the BLND token by 30% of it, with v2's allowance expiration (`LEDGER_BUMP_USER`), adds 70% to its depositors' schedule (below) and zeroes the allotment. Once a day per pool. |
| `distribute` | none | `i128`, BLND split | nobody | Calls the emitter's `distribute`, then splits the BLND due since the last distribution across the zone as v2's `distribute` (below). The first call after the swap only stores the emitter's time for the distributor. Does nothing before the swap. |
| `drop` | none | none | nobody | Calls the emitter's `drop` with the drop list, as v2's backstop did. The emitter takes the distributor's authorization, mints the listed amounts, at most 50M BLND in all, and drops once. |
| `add_reward` | `to_add: Address, to_remove: Option<Address>` | none | nobody | Calls `factory.get_backstop(to_add)`, which panics unless `to_add` is a standard deployment, then adds `to_add` as v2's `add_reward`: its backstop's `total_assets` must meet the admission threshold (below), `distribute` must have run within the hour unless the zone is empty, and in a full zone `to_add` replaces `to_remove` only if its backstop's `total_assets` is higher. |
| `remove_reward` | `to_remove: Address` | none | nobody | Removes `to_remove` as v2's `remove_reward`: only once its backstop's `total_assets` no longer meets the admission threshold, and only if `distribute` has run within the hour. |
| `reward_zone` | none | `Vec<Address>` | nobody | The pools in the zone. |

Amounts of BLND and of LP tokens have 7 decimals; shares are amounts of the backstop's share token. Kept functions keep v2's names.

### Eligibility

`add_reward` admits a pool when `factory.get_backstop(pool)` returns and the backstop's `total_assets` meets the admission threshold. The threshold is v2's product constant of the BLND and USDC behind the backstop's LP tokens, computed from `total_assets` and the Comet pool's balances, against a value set before deployment. The pool's own `min_backstop` plays no part.

### Reports

`report` calls `factory.get_backstop(pool)`, which returns the backstop of a `Standard` deployment and panics for any other address (spec/factory.md), and requires that backstop's authorization, which the invoking backstop contract gives and nothing else can. The distributor stores no binding between a pool and its backstop. A custom-route pool's backstop cannot report: the factory call panics. Reports are accepted from a standard backstop's first balance change, before its pool is in the zone. Outside the zone a pool is allotted nothing; its depositors' shares are still tracked, so the per-depositor records are complete when the pool is admitted.

A report updates first and stores second: the pool's index and `user`'s accrual at the stored shares, then `shares` replaces `user`'s stored shares and the pool's share total moves by the difference. The share total is the sum of the depositors' reported shares, the pool's unqueued shares.

### Distribution

`distribute` splits the BLND due since the last distribution, 1 BLND per second, across the pools in the zone as v2's `distribute`: it reads each pool's `unqueued_assets` from the backstop `factory.get_backstop(pool)` returns, and each pool's part is its amount over the zone's total, floor-rounded. The part is added to the pool's allotment, v2's `RzEmissions`.

### Depositors

Each pool has v2's backstop emission data: a rate, the time it ends and an index of BLND per unqueued share, scaled `SCALAR_14`. A gulp adds the pool's 70% to what the rate has not yet emitted and spreads the sum evenly over the next 7 days, as v2's `set_backstop_emission_eps`. Every report, claim and gulp for the pool first raises the index by the rate times the seconds since its last update, up to the end, over the pool's share total, as v2's `update_emission_data`; while the share total is zero the index stays, and the BLND scheduled for that time stays in the distributor. A depositor named in the call then accrues their stored shares times the index's movement since their last update, as v2's `update_user_emissions`. Queued shares earn nothing: no depositor's reported shares include them.

### Claims

`claim(from, pools, to)` updates `from` in each listed pool in order, as a report does, takes the accrued BLND of each and sets it to zero, and transfers the sum to `to`. One transfer per claim. The BLND is paid as BLND; nothing is swapped or redeposited. A listed pool need not be in the zone.

### Pool pull

`gulp_pool_emissions(pool)` takes the pool's allotment and splits it as v2's `gulp_emissions`: 30% raises the pool's allowance on the BLND token, 70% goes to the depositors' schedule. No BLND moves at a gulp. A pool outside the zone still gulps what it was allotted. Once a day per pool, as v2.

### Emitter handover

The emitter is v1's contract, kept as it is; `emitter/README.md` in blend-contracts-v2 points to its source (`emitter/src/emitter.rs` and `emitter/src/backstop_manager.rs` in blend-contracts v1). A backstop swap compares the candidate's balance of the emitter's stored backstop token, the old Comet LP token, with the current recipient's; the candidate must hold more.

1. The distributor is deployed. The YieldBlox Security Council transfers the old Comet LP shares it holds to the distributor.
2. Anyone calls `queue_swap_backstop(distributor, token)` on the emitter; the unlock time is 31 days later. `token` is what the emitter will store as its backstop token after the swap (see Open points).
3. During the 31 days anyone may call `cancel_swap_backstop`; it succeeds only if the distributor's balance has stopped exceeding the current recipient's.
4. After the unlock, anyone calls `swap_backstop`. The emitter pays the current recipient one last time, records the distributor and `token`, and sets its time for the distributor to that timestamp.
5. Anyone calls `distribute`. The distributor stores the emitter's time for it, the swap's, as its last distribution and splits from there.

The transfer redirects emissions only; it redeems no old LP shares and moves no depositor position.

## Types

| Type | Fields | Content |
|---|---|---|
| `RzEmissions` | `accrued: i128, last_time: u64` | v2's: BLND allotted by `distribute` and not yet gulped; the time of the last gulp |
| `BackstopEmissionData` | `expiration: u64, eps: u64, index: i128, last_time: u64` | v2's: when the schedule ends; its BLND per second, at 14 decimals; the pool's index; the time of the index's last update |
| `UserEmissionData` | `index: i128, accrued: i128` | v2's: the pool's index at the depositor's last update; BLND accrued and not claimed |

## Storage

| Key | Type | Content |
|---|---|---|
| `Emitter`, `BlndToken`, `UsdcToken`, `LpToken`, `Factory` | instance | the constructor's dependencies |
| `DropList` | instance, `Vec<(Address, i128)>` | the constructor's drop list |
| `RZ` | persistent, `Vec<Address>` | the pools in the zone, at most `MAX_RZ_SIZE`, v2's constant |
| `LastDist` | persistent, `u64` | the last distribution; unset until the first `distribute` after the swap |
| `PoolShares(pool)` | persistent, `i128` | the pool's share total |
| `RzEmis(pool)` | persistent, `RzEmissions` | the pool's allotment |
| `BEmisData(pool)` | persistent, `BackstopEmissionData` | the depositors' schedule and index |
| `UserShares(pool, user)` | persistent, `i128` | the depositor's reported shares |
| `UEmisData(pool, user)` | persistent, `UserEmissionData` | the depositor's accrual |

Nothing is keyed by a backstop address. The distributor holds every BLND the emitter has minted to it that no depositor, supplier or borrower has claimed, and the old Comet LP shares from the handover. Allowances live on the BLND token; the distributor stores none.

## Events

| Event | Topics | Data |
|---|---|---|
| `distribute` | `["distribute"]` | `amount`, split across the zone |
| `rw_zone_add` | `["rw_zone_add"]` | `[to_add, to_remove]` |
| `rw_zone_remove` | `["rw_zone_remove"]` | `to_remove` |
| `claim` | `["claim", from]` | `[to, amount]` |
| `report` | `["report", pool]` | `[user, shares]` |
| `gulp_pool_emissions` | `["gulp_pool_emissions", pool]` | `[backstop_emissions, pool_emissions]` |

`distribute`, `rw_zone_add`, `rw_zone_remove` and `claim` keep v2's topics.

## Errors

| Error | Code | When |
|---|---|---|
| `NegativeAmountError` | 8 | a negative amount in a report |
| `BadRequest` | 1000 | a gulp within a day of the last, a `distribute` within 5 seconds of the last or with an empty zone, a zone change more than an hour after the last `distribute`, a pool added twice, a `remove_reward` of a pool still above the threshold, an empty claim list, a pool listed twice or a claimed pool without emission data |
| `InvalidRewardZoneEntry` | 1002 | a pool below the threshold at `add_reward`, a replacement whose backstop's `total_assets` is not higher, or a `to_remove` outside the zone |
| `RewardZoneFull` | 1009 | a full zone with no `to_remove` |
| `NothingToClaim` | new, in v2's range | a claim whose sum is zero |

All but `NothingToClaim` are v2's backstop codes for the same conditions. A report or `add_reward` for a pool that is not a standard deployment fails with the factory's `NotStandardDeployment` (spec/factory.md); a report without the backstop's authorization is a host error.

## Invariants

- The reward zone never exceeds `MAX_RZ_SIZE`.
- `distribute` splits at most the BLND due since the last distribution, 1 BLND per second from the swap on, only to pools in the zone and by their backstops' `unqueued_assets`. Nothing accrues for time before the swap.
- Every index never decreases. A pool's index moves only by its schedule, over its unqueued shares.
- At the end of every call a pool's share total equals the sum of its depositors' stored shares, so its depositors together never accrue more than its schedule emits.
- The BLND allotted, scheduled and accrued never exceeds what the emitter has minted to the distributor. The distributor's balance covers every allotment, every schedule's remainder, every unclaimed accrual and every allowance it has raised.
- Each gulp splits the pool's allotment 70% to its depositors and 30% to the pool. Queued shares earn nothing.
- Only a standard deployment of the factory reports or enters the zone; the factory's record decides, on every report and at `add_reward`, never a caller-supplied address. The distributor stores no pool–backstop binding and nothing under a backstop address.
- `min_backstop` never affects eligibility.
- `distribute` reads each backstop in the zone; no call iterates the depositors, and a claim touches the pools the caller lists and no other. A report calls only the factory, and the distributor never calls a pool.
- BLND leaves the distributor only by `claim`, to the `to` it names, and by `transfer_from` against an allowance a gulp raised. The distributor never transfers an LP token of either Comet pool.

## Differences from v2

Compared with blend-contracts-v2 at the audited mainnet commit.

- Location. The reward zone, `distribute` and the pool pull come from `backstop/src/emissions/manager.rs`; the depositor index from `backstop/src/emissions/distributor.rs` and the claim from `backstop/src/emissions/claim.rs`. All of it is now one contract. v2 kept per-depositor emission data in the backstop and settled it from the backstop's own share records; v3's backstop keeps none and reports share balances instead.
- Balances. v2's emission logic read `PoolBalance` and `UserBalance` from the backstop's own storage. The distributor takes each depositor's shares from reports and reads the backstop's `unqueued_assets` at `distribute` and its `total_assets` at zone changes. v2 stopped a pool's index while it had no shares at all, which gave that time's BLND to the next depositor; the distributor lets the time pass.
- Emitter. v2's `distribute` read `get_last_distro` and left the emitter's `distribute` to anyone; v3's `distribute` calls it first.
- Binding. v2's backstop kept the pool binding itself: one backstop served every pool and asked `pool_factory.is_pool` at a pool's first deposit or donation (`require_is_from_pool_factory`, `backstop/src/backstop/pool.rs`). v3 keeps the pool–backstop binding in the factory; the distributor reads `get_backstop(pool)` on every report, at `distribute` and at zone changes and stores none.
- Threshold. v2's `is_pool_above_threshold` (`backstop/src/backstop/pool.rs`) took BLND and USDC from `load_pool_backstop_data`, which read the Comet pool; the distributor computes them from the backstop's `total_assets` and the Comet pool. The threshold was a constant in that function and is set before deployment; the zone size stays v2's `MAX_RZ_SIZE` (`backstop/src/constants.rs`).
- Pull. v2's `gulp_emissions(pool)` is `gulp_pool_emissions(pool)`, with the same split, schedule, event data and once-a-day rule.
- Claims. v2's `claim(from, pool_addresses, min_lp_tokens_out)` (`backstop/src/emissions/claim.rs`) swapped the BLND into LP tokens through the Comet pool and redeposited them for `from`; v3's `claim(from, pools, to)` keeps the pool list and pays BLND to `to`, touching no backstop.
- Pool side. v2's pool module (`pool/src/emissions/`) called `backstop.gulp_emissions` and drew its allowance from the backstop; it now calls `distributor.gulp_pool_emissions` and draws from the distributor.
- Backfill. v2's `distribute` could backfill emissions for the period before its swap completed (`MAX_BACKFILLED_EMISSIONS`); the distributor has no backfill.

## Open points

1. Guarding the emitter after the handover (README open question 1). The backstop swap compares balances of one token at one address, and after the swap the v2.1 Comet LP tokens sit in the backstops, not in the distributor. What the distributor names as its token decides who can take the emitter later and how a successor is installed, including how the distributor itself is replaced.
2. The emitter drop (README open question 2). Who is on the drop list and for how much.
