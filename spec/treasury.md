# Treasury

## Purpose

The treasury holds the protocol rate, names the payout token that protocol interest auctions are paid in, and holds what those auctions collect from every pool. It is one contract for the whole protocol, owned, with no upgrade entry point, and every pool reads it directly.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `owner: Address, rate: u32, payout_token: Address, price_source: Address` | none | deployer | Sets the owner, the rate, the payout token and its price source. Fails with `InvalidRate` if `rate` exceeds `MAX_PROTOCOL_RATE`. |
| `get_rate` | none | `u32` | nobody | The rate. |
| `get_token` | none | `Address` | nobody | The payout token. |
| `get_price` | none | `AssetPrice` | nobody | The payout token's price from the price source, in the source's decimals. Fails with `InvalidPrice` when the source has no price for the token, the price is not positive, or it is older than 24 hours. |
| `set_rate` | `rate: u32` | none | owner | Sets the rate. Fails with `InvalidRate` above `MAX_PROTOCOL_RATE`. |
| `set_payout_token` | `token: Address, price_source: Address` | none | owner | Sets the payout token and the source that prices it, together. Fails with `InvalidPrice` if the source reports no valid price for the token. Open auctions keep their bid token. |
| `withdraw` | `token: Address, to: Address, amount: i128` | none | owner | Transfers `amount` of `token` from the treasury to `to`. Any token. |
| Ownership | `Ownable`: `get_owner`, `transfer_ownership`, `accept_ownership`, `renounce_ownership` | as OpenZeppelin | as OpenZeppelin | OpenZeppelin's implementation. `renounce_ownership` fails with `RateNotZero` while the rate is above zero. |

The price source is a SEP-40 oracle, the pool's own price interface. The treasury returns the source's `lastprice` for the payout token with the source's `decimals`, quoted in the base the pool oracles use.

Rates are `u32` with 7 decimals, as the backstop take rate is. `MAX_PROTOCOL_RATE` is a constant; its value is set before deployment.

### What a pool does with it

A pool holds the treasury address from deploy, set by the factory and never changed. When a reserve accrues interest the pool calls `get_rate`, clamps the result to its own `MAX_PROTOCOL_RATE` and credits `accrued × rate`, rounded down, to the reserve's `protocol_credit` in the borrowed asset. `backstop_credit` is credited at the backstop take rate in the same step, and suppliers receive the remainder through `b_rate`. The pool keeps no copy of the rate: interest is accrued at the rate the treasury holds when the reserve is next touched, including interest that built up between touches. A pool with a backstop take rate of zero accrues `protocol_credit` like any other.

### How a protocol interest auction pays it

The pool sells `protocol_credit` in a protocol interest auction, the twin of v2's backstop interest auction. The lot is the accrued credit of the listed reserves. The bid is the payout token, sized at 1.2 times the lot's value in the pool's oracle divided by `get_price`, rescaled to the oracle's decimals, and it falls along v2's auction curve. Anyone can create one once the lot is worth the minimum a backstop interest auction needs; one runs per pool at a time. At fill the pool transfers the bid from the filler to the treasury under the filler's authorization, transfers the lot from the pool to the filler and deducts each delivered lot amount from its reserve's `protocol_credit`. The treasury has no entry point on this path: it receives a plain token transfer and cannot block or alter a fill. If the payout token changes while an auction is open, the auction keeps its bid token and anyone may delete it (spec/pool.md). Credit below the auction minimum stays in its pool.

## Types

| Type | Fields | Content |
|---|---|---|
| `AssetPrice` | `price: i128, decimals: u32` | the price of one payout token in the pool oracles' base, in `decimals` |

## Storage

| Key | Type | Content |
|---|---|---|
| `Rate` | instance, `u32` | the protocol rate, 7 decimals, at most `MAX_PROTOCOL_RATE` |
| `PayoutToken` | instance, `Address` | the token protocol interest auctions are paid in |
| `PriceSource` | instance, `Address` | the oracle the payout token's price is read from |
| owner, pending owner | OpenZeppelin `Ownable` entries | |

Every entry point extends the instance TTL. `get_rate` and `get_token` read instance storage and nothing else; they fail only on an archived instance, which any account can restore. `MAX_PROTOCOL_RATE` is compiled into the treasury and the pool with the same value; the pool's copy bounds the fee, the treasury's keeps `get_rate` equal to the rate pools apply. Collected fees are the contract's token balances, with no record of which pool paid what. No role exists besides the owner.

## Events

| Event | Topics | Data |
|---|---|---|
| `set_rate` | `["set_rate"]` | the new rate |
| `set_payout_token` | `["set_payout_token"]` | the token and its price source |
| `withdraw` | `["withdraw"]` | `token`, `to`, `amount` |
| `ownership_transfer`, `ownership_transfer_completed`, `ownership_renounced` | as OpenZeppelin | as OpenZeppelin |

A fill publishes no treasury event; the token's transfer event and the pool's fill event record it.

## Errors

| Error | Code | When |
|---|---|---|
| `InvalidRate` | new | a rate above `MAX_PROTOCOL_RATE` in the constructor or `set_rate` |
| `InvalidPrice` | new | the price source has no price for the payout token, or the price is not positive or too old |
| `RateNotZero` | new | `renounce_ownership` while the rate is above zero |
| OpenZeppelin `OwnableError`, `RoleTransferError` | as OpenZeppelin | the ownership functions |

## Invariants

- The stored rate never exceeds `MAX_PROTOCOL_RATE`, and a pool clamps whatever it reads to the same constant. Per accrual, `protocol ≤ accrued × MAX_PROTOCOL_RATE`.
- The caps on the protocol rate and on the backstop take rate, both constants in pool code, together take less than the whole of the interest. The suppliers' remainder is never negative.
- A fill pays only the treasury address the pool was deployed with. Neither the pool admin, a hook, the filler nor the treasury owner can redirect it.
- The lot of a protocol interest auction is `protocol_credit` only. The treasury never receives supplier or backstop funds.
- The pool calls the treasury only through the three getters, and the treasury calls no pool. A failing `get_price` stops new protocol interest auctions and nothing else; the credit keeps accruing in the pool.
- After `renounce_ownership` the rate is zero and stays zero, the payout token and price source never change, and nothing leaves the treasury. Bids filled after that, for credit accrued before the rate reached zero, still arrive at the treasury.

## Differences from v2

v2 has no treasury and no protocol share. In the audited v2 code (`blend-contracts-v2`):

- `pool/src/pool/reserve.rs`, `Reserve::accrue`: one rate, `bstop_rate`, into `backstop_credit`, the remainder to suppliers through `b_rate`. v3 adds `protocol_credit` at the treasury's rate in the same step.
- `pool/src/pool/config.rs`, `require_valid_pool_config`: `bstop_rate` must be below one, nothing more. v3 caps it by a constant and the protocol rate by a second, the two together below one.
- `pool/src/auctions/backstop_interest_auction.rs`: the bid is sized with `backstop.pool_data().token_spot_price` and the fill goes through `backstop.donate`. v3 sizes that bid with `backstop.asset_price()` and fills it by a direct transfer to the backstop (spec/pool.md), and adds a second interest auction type that sizes its bid with `treasury.get_price` and fills by a direct transfer to the treasury.
- `pool/src/auctions/auction.rs`, `AuctionType`: three types. v3 adds the protocol interest auction as a fourth (spec/pool.md).
- `pool/src/contract.rs`, `__constructor`, and `pool-factory/src/storage.rs`, `PoolInitMeta`: a pool is deployed with its backstop and BLND addresses. v3's factory also bakes the treasury address into every pool (spec/factory.md).
- `pool/src/pool/pool.rs`, `load_price`: a SEP-40 `lastprice` read with a sign and age check. The treasury reads its price source the same way; the oracle interface is unchanged.
