# Pool hooks

## Purpose

A hook is an optional contract, one per pool and fixed when the pool is deployed, that the pool calls once at the end of every batch with the batch and the state that resulted from it. It can reject the batch by panicking, which reverts the transaction, and it cannot change what the pool did with the batch.

## Interface

| Function | Arguments | Returns | Who must authorize | Effect |
|---|---|---|---|---|
| `on_submit` (hook) | `from: Address, spender: Address, to: Address, requests: Vec<Request>, reserves: Vec<Reserve>, positions: Positions` | none | None; the pool is the caller. Any authorization the hook's own calls need comes from the transaction the user signed. | Receives the batch's parties, the batch as submitted, every reserve the batch changed and `from`'s final positions. A return accepts the batch; a panic rejects it and reverts the whole transaction. |

A hook may expose any other functions it needs, such as list management under its own owner; the pool calls only `on_submit`.

**Arguments.**

- `from`, `spender`, `to`: the parties of the `submit` call. For `flash_loan`, `spender` and `to` are `from`.
- `requests`: the batch exactly as submitted, every request type included. For `flash_loan`, the request list that followed the loan; the loaned asset is among `reserves` and any liability left from it is in `positions`.
- `reserves`: every reserve the batch loaded for writing, in the order the batch first touched them, as each stands after interest accrual and after every request: the assets of the requests and, for an auction fill, the reserves of the lot and the bid.
- `positions`: `from`'s positions after the batch, the value `submit` returns.

`Request`, `Reserve` and `Positions` are the pool's own types, already exposed by `submit`, `get_reserve` and `get_positions`. A hook needs no new type.

**When it is called.** Once per call to `submit`, `submit_with_allowance` or `flash_loan`, whatever the batch contains: a batch that only withdraws or repays, and an empty batch, call it too. The call comes last, after every request has been applied, after the batch's checks have passed, the health check included, and after the tokens have moved; for a flash loan, after the receiver contract has run as well. No other pool function calls the hook: admin functions, `gulp`, `update_status`, `bad_debt`, `new_auction` and `clawback` run without it. The backstop never calls a hook.

**What a panic does.** A panic in the hook, for any reason, reverts the transaction. Every storage write and every token transfer the batch made is undone, and the hook's own error reaches the caller of `submit`. A hook that exceeds the transaction's resource limits fails the batch the same way.

**What the hook cannot do.** The pool is on the call stack while `on_submit` runs and Soroban forbids re-entry, so the hook cannot call the pool. The arguments are copies, so the hook cannot change amounts, accounting or transfers. That is the only guarantee the pool gives about a hook. Everything else is the hook's code: it can read other contracts, keep its own state, and make calls under the authorization the user signed for the transaction, a fee transfer included. It can refuse any batch, including one that only exits. The pool does not identify itself to the hook beyond being the caller; a hook that keeps state across calls checks that its caller is its pool.

**Trust model.** The hook address is set by the factory at deploy, written once into the pool, published in the factory's `deploy` event and returned by `get_hook`. No function changes it. The hook's code is on-chain like any contract's. The maintainers publish verified builds of known hooks; the UI shows a pool's hook on the pool's page and warns when it is not one of them. A hook is part of a pool's trust assumptions, like its admin and its oracle.

**Example policies.**

| Policy | Rule in `on_submit` |
|---|---|
| Allowlist | `from`, `spender` and `to` must be on the hook's list; otherwise panic. Whether a delisted address may still exit is the hook's choice. |
| Gated liquidator | If `requests` holds an auction fill, `from` must be on the list; other batches pass. |
| Supplier-only | A `Supply` request must come from one configured address. With a fee vault as that address, depositors enter through the vault and the vault takes the operator's share of their interest. `SupplyCollateral` and every other request pass. |

These are examples of what `on_submit` can express. None is part of v3 and none is promised: a pool admin writes or chooses a hook and publishes a verifiable build of it. Lock-ups and limits follow the same pattern: the hook reads `requests`, `reserves` and `positions` and panics when its rule fails.

## Storage

| Key | Type | Content |
|---|---|---|
| `Hook` (pool) | instance, `Option<Address>` | the hook, written by the constructor from the factory's argument and never changed |

The pool stores nothing else for the hook. What the hook stores, lists, owners or counters, is its own; the pool neither reads nor writes it.

## Events

The pool emits no event for a hook call, accepted or rejected. The factory's `deploy` event carries the hook address, `None` when there is none. A hook may emit its own events; the pool defines none for it.

## Errors

The pool adds no error code for hooks. A rejection is the hook's own panic, and its error code reaches the caller of `submit` unchanged, so an integrator sees the hook's code, not a pool code.

## Invariants

- The hook address is written once by the pool constructor; no function changes it; `get_hook` returns it.
- With no hook, the pool makes no call.
- Exactly one hook call per `submit`, `submit_with_allowance` and `flash_loan`, after every request, every check and every transfer of the batch; never from any other function.
- The hook sees the final state of the batch and only that; intermediate states inside a batch are not observable.
- Nothing the hook receives or does can alter what the pool wrote or transferred in the batch.
- A hook panic leaves no trace: the transaction reverts in full.
- Which batches pass, and what the hook does under the user's authorization, are properties of the hook's code, not of the pool.
- The backstop has no hook.

## Differences from v2

v2 has no hook. In `pool/src/pool/submit.rs`, `execute_submit` and `execute_submit_with_flash_loan` build the batch's actions from the requests (`pool/src/pool/actions.rs`), validate them in `validate_submit`, move the tokens, then store the cached reserves and `from`'s positions. v3 adds one call after the transfers, passing the parties, the requests, the reserves the batch cached for writing (`Pool::reserves_to_store` in `pool/src/pool/pool.rs`) and `from`'s positions. The pool constructor in `pool/src/contract.rs` gains a `hook: Option<Address>` argument, the last (spec/pool.md); instance storage in `pool/src/storage.rs` gains the `Hook` key beside `Backstop`; `get_hook` is added to the getters.