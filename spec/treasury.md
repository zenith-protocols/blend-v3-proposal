# Treasury

## Purpose

The treasury holds the protocol rate and the USDC that protocol interest auctions collect from every pool. It sets aside a share of each payment and auctions it for BLND, which is burned. It is one contract for the whole protocol, owned, with no upgrade entry point, and every pool reads it directly.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `owner: Address, rate: u32, split: u32, blnd: Address, comet: Address` | none | deployer | Sets the owner, the rate, the buyback split, BLND and the v1.1 Comet pool. Reads USDC from the pool's `get_tokens` as the token that is not `blnd`. Fails with `InvalidRate` if `rate` exceeds `MAX_PROTOCOL_RATE`, with `InvalidSplit` if `split` exceeds 100% and with `InvalidComet` if `blnd` is not one of the pool's two tokens. |
| `get_rate` | none | `u32` | nobody | The rate. |
| `get_token` | none | `Address` | nobody | USDC. |
| `set_rate` | `rate: u32` | none | owner | Sets the rate. Fails with `InvalidRate` above `MAX_PROTOCOL_RATE`. |
| `get_split` | none | `u32` | nobody | The buyback split. |
| `set_split` | `split: u32` | none | owner | Sets the buyback split. It applies to later donations. Fails with `InvalidSplit` above 100%. |
| `donate` | `from: Address, amount: i128` | none | `from` | Transfers `amount` of USDC from `from` to the treasury and adds `amount × split`, rounded down, to `Buyback`. |
| `new_auction` | none | `BuybackAuction` | nobody | Starts a buyback auction of all of `Buyback` (below). Fails with `AuctionInProgress` while one is open and with `LotTooSmall` below the minimum. |
| `fill_auction` | `filler: Address` | none | `filler` | Fills the open buyback auction in full (below). Fails with `NoAuction` when none is open. |
| `get_auction` | none | `BuybackAuction` | nobody | The open buyback auction. Fails with `NoAuction` when none is open. |
| `withdraw` | `token: Address, to: Address, amount: i128` | none | owner | Transfers `amount` of `token` from the treasury to `to`. Any token. Fails with `InsufficientFunds` above the treasury's balance of `token`, less `Buyback` and the open buyback auction's lot when `token` is USDC. |
| Ownership | `Ownable`: `get_owner`, `transfer_ownership`, `accept_ownership`, `renounce_ownership` | as OpenZeppelin | as OpenZeppelin | OpenZeppelin's implementation. `renounce_ownership` fails with `RateNotZero` while the rate is above zero and with `InvalidSplit` while the split is below 100%. |

Rates and the split are `u32` with 7 decimals, as the backstop take rate is. `MAX_PROTOCOL_RATE` is a constant, 5%.

### What a pool does with it

A pool holds the treasury address from deploy, set by the factory and never changed. When a reserve accrues interest the pool calls `get_rate`, clamps the result to its own `MAX_PROTOCOL_RATE` and credits `accrued × rate`, rounded down, to the reserve's `protocol_credit` in the borrowed asset. `backstop_credit` is credited at the backstop take rate in the same step, and suppliers receive the remainder through `b_rate`. The pool keeps no copy of the rate: interest is accrued at the rate the treasury holds when the reserve is next touched, including interest that built up between touches. A pool with a backstop take rate of zero accrues `protocol_credit` like any other.

### How a protocol interest auction pays it

The pool sells `protocol_credit` in a protocol interest auction, the twin of v2's backstop interest auction. The lot is the accrued credit of the listed reserves. The bid is USDC, sized at 1.2 times the lot's value in the pool's oracle, with one unit of the oracle's base worth one USDC, and it falls along v2's auction curve. Anyone can create one once the lot is worth the minimum a backstop interest auction needs. One runs per pool at a time. At fill the pool calls `donate(filler, bid)` under the filler's authorization, transfers the lot from the pool to the filler and deducts each delivered lot amount from its reserve's `protocol_credit`.

### Buyback auction

`new_auction` sells all of `Buyback` for BLND. It requires the lot to be at least 200 USDC, the backstop interest auction's minimum. The bid is 1.2 times the lot divided by BLND's spot price in the v1.1 Comet pool, four times its USDC balance over its BLND balance, without fees. `Buyback` is zeroed and its amount becomes the lot. One auction runs at a time.

The auction starts the ledger after `new_auction`. Lot and bid follow v2's auction curve: the lot grows from 0% to 100% over 200 ledgers with the bid whole, then the bid falls linearly to zero 400 ledgers after the start. `fill_auction` burns the scaled bid from the filler under the filler's authorization, transfers the scaled lot to the filler and deletes the auction. There are no partial fills. A zero bid burns nothing. The undelivered lot returns to `Buyback`.

## Types

| Type | Fields | Content |
|---|---|---|
| `BuybackAuction` | `lot: i128, bid: i128, block: u32` | the USDC lot, the BLND bid at the start and the start ledger |

## Storage

| Key | Type | Content |
|---|---|---|
| `Rate` | instance, `u32` | the protocol rate, 7 decimals, at most `MAX_PROTOCOL_RATE` |
| `Split` | instance, `u32` | the buyback split, 7 decimals, at most 100% |
| `Blnd` | instance, `Address` | the BLND token |
| `Usdc` | instance, `Address` | the USDC token |
| `Comet` | instance, `Address` | the v1.1 Comet BLND–USDC pool |
| `Buyback` | instance, `i128` | USDC set aside for the next buyback auction |
| `Auction` | instance, `BuybackAuction` | the open buyback auction |
| owner, pending owner | OpenZeppelin `Ownable` entries | |

Every entry point extends the instance TTL. `get_rate` and `get_token` read instance storage and nothing else; they fail only on an archived instance, which any account can restore. `MAX_PROTOCOL_RATE` is compiled into the treasury and the pool with the same value; the pool's copy bounds the fee, the treasury's keeps `get_rate` equal to the rate pools apply. Collected fees are the contract's token balances, with no record of which pool paid what. No role exists besides the owner.

## Events

| Event | Topics | Data |
|---|---|---|
| `set_rate` | `["set_rate"]` | the new rate |
| `withdraw` | `["withdraw"]` | `token`, `to`, `amount` |
| `set_split` | `["set_split"]` | the new split |
| `donate` | `["donate", from]` | `amount`, the part added to `Buyback` |
| `new_auction` | `["new_auction"]` | the `BuybackAuction` |
| `fill_auction` | `["fill_auction", filler]` | the lot and the BLND burned |
| `ownership_transfer`, `ownership_transfer_completed`, `ownership_renounced` | as OpenZeppelin | as OpenZeppelin |

## Errors

| Error | Code | When |
|---|---|---|
| `InvalidRate` | new | a rate above `MAX_PROTOCOL_RATE` in the constructor or `set_rate` |
| `InvalidComet` | new | `blnd` is not one of the Comet pool's two tokens in the constructor |
| `RateNotZero` | new | `renounce_ownership` while the rate is above zero |
| `InvalidSplit` | new | a split above 100% in the constructor or `set_split`, or `renounce_ownership` while the split is below 100% |
| `InsufficientFunds` | new | a `withdraw` above what `withdraw` may take |
| `AuctionInProgress` | new | `new_auction` while an auction is open |
| `LotTooSmall` | new | `new_auction` with `Buyback` below the minimum |
| `NoAuction` | new | `fill_auction` or `get_auction` with no open auction |
| OpenZeppelin `OwnableError`, `RoleTransferError` | as OpenZeppelin | the ownership functions |

## Invariants

- The stored rate never exceeds `MAX_PROTOCOL_RATE`, and a pool clamps whatever it reads to the same constant. Per accrual, `protocol ≤ accrued × MAX_PROTOCOL_RATE`.
- The caps on the protocol rate and on the backstop take rate, both constants in pool code, together take less than the whole of the interest. The suppliers' remainder is never negative.
- A fill pays only the treasury address the pool was deployed with. Neither the pool admin, a hook, the filler nor the treasury owner can redirect it.
- The lot of a protocol interest auction is `protocol_credit` only. The treasury never receives supplier or backstop funds.
- The pool calls the treasury only through `get_rate`, `get_token` and `donate`, and the treasury calls no pool. A failing `donate` stops protocol interest auctions and nothing else. The credit keeps accruing in the pool.
- `Buyback` and an open auction's lot leave the treasury only through `fill_auction`. The owner cannot withdraw them.
- A buyback auction's bid reaches zero 400 ledgers after it starts, so no auction blocks the next.
- After `renounce_ownership` the rate is zero and stays zero, the split is 100% and never changes, and nothing leaves the treasury except buyback lots. Bids filled after that, for credit accrued before the rate reached zero, still arrive at the treasury.

## Differences from v2

v2 has no treasury and no protocol share. In the audited v2 code (`blend-contracts-v2`):

- `pool/src/pool/reserve.rs`, `Reserve::accrue`: one rate, `bstop_rate`, into `backstop_credit`, the remainder to suppliers through `b_rate`. v3 adds `protocol_credit` at the treasury's rate in the same step.
- `pool/src/pool/config.rs`, `require_valid_pool_config`: `bstop_rate` must be below one, nothing more. v3 caps it by a constant and the protocol rate by a second, the two together below one.
- `pool/src/auctions/backstop_interest_auction.rs`: the bid is sized with `backstop.pool_data().token_spot_price` and the fill goes through `backstop.donate`. v3 sizes that bid with `backstop.asset_price()` and fills it by a direct transfer to the backstop (spec/pool.md), and adds a second interest auction type that sizes its bid in USDC and fills through `treasury.donate`.
- `pool/src/auctions/auction.rs`, `AuctionType`: three types. v3 adds the protocol interest auction as a fourth (spec/pool.md).
- `pool/src/contract.rs`, `__constructor`, and `pool-factory/src/storage.rs`, `PoolInitMeta`: a pool is deployed with its backstop and BLND addresses. v3's factory also bakes the treasury address into every pool (spec/factory.md).
- The buyback auction is new. It takes v2's interest-auction minimum and v2's auction curve (`pool/src/auctions/auction.rs`, `scale_auction`).
