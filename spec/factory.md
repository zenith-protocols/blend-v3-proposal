# Pool factory

## Purpose

The factory deploys a lending pool bound to a backstop: on the standard route it deploys the backstop itself from fixed code, on the custom route it binds one the pool admin deployed. It bakes the treasury, the distributor and the optional hook into every pool and records each pool with its route, which the distributor reads to tell a verified standard deployment from a recognized pool.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `__constructor` | `pool_init_meta: PoolInitMeta` | none | the factory's deployer | Stores the two code hashes, the four addresses and the standard backstop's decimals offset; `backstop_token` is the v2.1 Comet LP token. No setter. |
| `deploy` | `admin: Address, name: String, salt: BytesN<32>, oracle: Address, backstop_take_rate: u32, max_positions: u32, min_collateral: i128, min_backstop: i128, share_name: String, share_symbol: String, hook: Option<Address>` | `(Address, Address)`, the pool and the backstop | `admin` | Standard route, below. |
| `deploy_custom` | `admin: Address, name: String, salt: BytesN<32>, oracle: Address, backstop_take_rate: u32, max_positions: u32, min_collateral: i128, min_backstop: i128, backstop: Address, hook: Option<Address>` | `(Address, Address)`, the pool and the backstop | `admin` | Custom route, below. |
| `is_pool` | `pool: Address` | `bool` | none | True for a pool the factory deployed by either route. |
| `get_backstop` | `pool: Address` | `Address` | none | The backstop of a `Standard` deployment; fails with `NotStandardDeployment` (1302) for any other address. |

Both routes reject, with `InvalidPoolInitArgs` (1300), a backstop take rate above the cap the pool enforces, `max_positions` outside v2's bounds, a negative `min_collateral` and a negative `min_backstop`. `min_backstop` is an amount of the backstop's token, the v2.1 Comet LP token on the standard route; zero is allowed. The oracle, the hook and the share token metadata pass through unchecked.

### Address derivation

The pool salt is `keccak256(salt ‖ admin)`, with the admin in its 56-byte string form, as in v2 (`pool-factory/src/pool_factory.rs`). The pool address follows from the factory's address and that salt, so it is known before the pool exists. One admin and salt give one address, and only that admin can deploy at it, since both deploy functions require the admin's authorization. A standard backstop's salt is `keccak256(pool_salt ‖ "backstop")`.

### Standard route

1. `deploy` requires `admin`'s authorization and checks the arguments.
2. It derives the pool salt and the pool address.
3. It deploys the backstop from `backstop_hash` with the derived pool address, `backstop_token`, `distributor`, `decimals_offset`, `share_name` and `share_symbol` ([spec/backstop.md](backstop.md)).
4. It deploys the pool from `pool_hash` at the derived address, passing the backstop's address.
5. It stores `Deployment(pool) = { backstop, route: Standard }` and emits `deploy`.

### Custom route

1. Before calling the factory, the pool admin computes the pool address and deploys a backstop that implements the pool-facing interface in [spec/backstop.md](backstop.md): the vault standard's `query_asset`, `total_assets` and `total_supply`, its `balance` of itself, `draw`, `asset_price` and `pool`, with that address as its pool.
2. `deploy_custom` requires `admin`'s authorization and checks the arguments.
3. It derives the pool address and calls `backstop.pool()`. If the result differs, it fails with `InvalidBackstop` (1301).
4. It deploys the pool from `pool_hash` at the derived address, passing the supplied backstop address.
5. It stores `Deployment(pool) = { backstop, route: Custom }` and emits `deploy`.

The factory does not read the backstop's code hash, token, price source or deposit rules, and never calls it again.

### What the pool receives

On both routes the pool's constructor receives `admin, name, oracle, bstop_rate, max_positions, min_collateral, min_backstop, backstop_id, treasury, distributor, blnd_id, hook`: the first seven and `hook` from the deploy call, `backstop_id` from the route, the rest from the factory's constructor. The pool stores the backstop, `min_backstop`, the treasury, the distributor, the BLND token and the hook once; no function changes them ([spec/pool.md](pool.md)). The pool cannot tell the routes apart.

## Types

| Type | Fields | Content |
|---|---|---|
| `PoolInitMeta` | `pool_hash: BytesN<32>, backstop_hash: BytesN<32>, treasury: Address, distributor: Address, backstop_token: Address, blnd_token: Address, decimals_offset: u32` | the code hashes, addresses and decimals offset every deployment uses |
| `Deployment` | `backstop: Address, route: Route` | a pool's backstop and the route that bound it |
| `Route` | `Standard` or `Custom` | the deploy route |

## Storage

| Key | Type | Content |
|---|---|---|
| `PoolMeta` | instance, `PoolInitMeta` | the constructor's struct |
| `Deployment(pool)` | persistent, `Deployment` | the backstop and route of every pool deployed |

Nothing is keyed by a backstop address. The hook and `min_backstop` are not stored: the `deploy` event carries the hook, and the pool exposes both through `get_hook` and `get_config`.

A `Standard` record is a verified standard deployment: pool from `pool_hash`, backstop from `backstop_hash` holding `backstop_token`, bound to each other by the factory. A `Custom` record is a recognized pool: the pool code and its bindings are the factory's; the backstop is whatever named the pool at deploy time.

The distributor calls `get_backstop(pool)` on every report, at `distribute` and at zone changes: a report must carry the authorization of the address returned, and only a pool the call returns for enters the reward zone ([spec/distributor.md](distributor.md)). No record, or a `Custom` one, makes the call fail, so the pool's backstop cannot report and the pool is ineligible whatever token its backstop holds.

## Events

| Event | Topics | Data |
|---|---|---|
| `deploy` | `["deploy"]` | `[pool: Address, backstop: Address, route: Route, hook: Option<Address>]` |

Emitted once per pool.

## Errors

| Error | Code | When |
|---|---|---|
| `InternalError` | 1 | defined and not raised, as in v2 |
| `AlreadyInitializedError` | 3 | defined and not raised, as in v2 |
| `InvalidPoolInitArgs` | 1300 | an argument check under Interface fails, `min_backstop` included |
| `InvalidBackstop` | 1301 | the custom route's backstop does not name the derived pool address |
| `NotStandardDeployment` | 1302 | `get_backstop` for an address without a `Standard` record |

`PoolFactoryError` keeps v2's codes (`pool-factory/src/errors.rs`); 1301 and 1302 are new. A missing authorization, or a call into a contract without `pool()`, is a host error, not a factory code.

## Invariants

- Every recorded pool was created from `pool_hash` with the factory's treasury, distributor and BLND token; its backstop, `min_backstop` and hook were written once by its constructor.
- One pool per address and one backstop per pool: the address is fixed by admin and salt, a contract deploys at it once, and the pool stores its backstop once. A custom backstop naming a pool address can be bound to that pool only.
- The deployer never picks the pool code, the standard backstop's code or token, the treasury or the distributor.
- Deploying needs only the admin's signature; there is no list of allowed deployers.
- The factory has no admin, no upgrade and no setter. New pool code, backstop code, token, treasury or distributor means a new factory; the distributor reads one factory.

## Differences from v2

Compared with `pool-factory/src` at the audited v2 commit:

- `PoolInitMeta` was `{ pool_hash, backstop, blnd_id }` (`storage.rs`): one shared backstop for every pool. It is now `{ pool_hash, backstop_hash, treasury, distributor, backstop_token, blnd_token, decimals_offset }`.
- `deploy` made one deployment (`pool_factory.rs`), the pool, with the shared backstop in its constructor, and returned the pool. It now makes two, backstop first, and returns both. `deploy_custom` and `get_backstop` are new.
- v2 stored `Contracts(pool) → true` (`storage.rs`), and its backstop asked `is_pool` before a pool's first deposit or donation (`backstop/src/backstop/pool.rs`, `require_is_from_pool_factory`). The record now carries the backstop and the route, the backstop no longer calls the factory, and the distributor reads the record on every report.
- The salt derivation is unchanged. The argument checks are v2's three plus `min_backstop`; the take-rate bound is the pool's cap, not v2's bound of one.
- The `deploy` event carried the pool address only (`events.rs`); it now carries the backstop, the route and the hook.
- The pool's constructor arguments changed accordingly ([spec/pool.md](pool.md)).

