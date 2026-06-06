## [M-1] `LPPlugin::deposit()` keeps unconsumed user funds on the plugin instead of refunding

### Summary

`deposit()` pulls `amount0` and `amount1` from the user, computes `actual0/actual1`, approves both amounts to `callback`, and calls:

solidity Copy

```scss
ILPCallback(callback).mint(recipient, tickLower, tickUpper, actual0, actual1);
```

However, it ignores the callback return values:

solidity Copy

```scss
returns (uint256 amount0Returned, uint256 amount1Returned, uint128 returnedLiquidity)
```

When the chosen range is out of the current price or the deposit is imbalanced, the pool may consume only one side (or partially consume one side). The unconsumed side remains on `LPPlugin`, but `deposit()` does not refund it to the user.

So user funds can become stranded in the plugin as idle balances that are not explicitly accounted for in share minting at deposit time.

### Impact

-   Users can lose immediate control over unconsumed assets after `deposit()`.
-   Plugin token balances can accumulate stranded funds over time.
-   Share minting/withdraw accounting may become economically inconsistent with the plugin's actual idle balances, creating fairness issues between participants.

### Proof of concept

### Summary

2.  Keep the current pool price below the selected range (e.g. `currentTick = 0`, deposit into `[500, 1000]`).
3.  User calls `deposit(recipient, 500, 1000, amount0=10e18, amount1=5000e18, minLPTokens=0, deadline=...)`.
4.  `LPPlugin` transfers both tokens from the user and approves both to `callback`.
5.  `LPCallback.mint()` computes liquidity for that out-of-range position; practically only one token side is needed by the pool for mint.
6.  Pool pulls only the owed side from `LPPlugin` via mint callback; the other side is left unused.
7.  `deposit()` continues, sends LP shares, but never refunds `actualX - amountXUsed` to the user.
8.  Check balances: user receives LP tokens but does not recover the unused token; `LPPlugin` balance for that token increases and remains there.

## [M-2] `withdraw()` uses stale fee snapshot, causing order-dependent fee redistribution

### Summary

`withdraw()` snapshots `fees0/fees1` from `pool.positions(...)` before calling `_collect(...)`:

solidity Copy

```kotlin
(uint256 liquidity,,, uint128 fees0, uint128 fees1) =
    IAlgebraPool(pool).positions(getPositionKey(address(this), tickLower, tickUpper));
```

Then `_collect(...)` performs `pool.burn(...)`, which is the operation that crystallizes pending fees into position accounting. However, the later `pool.collect(...)` limit still uses the pre-burn fee snapshot:(In algebra, use burn(0) to update fees)

solidity Copy

```csharp
uint128(Math.mulDiv(params.lpTokensToBurn, params.fees0, params.totalLPSupply))
uint128(Math.mulDiv(params.lpTokensToBurn, params.fees1, params.totalLPSupply))
```

So the withdrawal amount is computed from stale fee values, while newer fees become available only after `burn`.

### Official documentation basis

The behavior above is directly aligned with Algebra’s official API docs:

-   `collect(...)` does **not** recompute earned fees; fee recomputation must happen via `mint` or `burn`.
-   `burn(...)` can be called with `amount = 0` to trigger fee recalculation, and fee withdrawal is still a separate `collect(...)` step.
-   `positions(...).fees0/fees1` are defined as values computed **as of the last mint/burn/poke** (i.e., an accounting snapshot, not a continuously refreshed live value).

Official references:

### Summary

-   Algebra docs, `IAlgebraPoolActions` (`collect` / `burn` semantics):  
    [https://docs.algebra.finance/algebra-integral-documentation/algebra-v1-technical-reference/contracts/api-reference-v2.0/v2.0-core/ialgebrapoolactions](https://docs.algebra.finance/algebra-integral-documentation/algebra-v1-technical-reference/contracts/api-reference-v2.0/v2.0-core/ialgebrapoolactions)
-   Algebra docs, `IAlgebraPoolState` (`positions` fields and "last mint/burn/poke"):  
    [https://github.com/cryptoalgebra/algebra-integral-docs/blob/main/integration-of-algebra-integral-protocol/contracts-api/Core/interfaces/pool/IAlgebraPoolState.md](https://github.com/cryptoalgebra/algebra-integral-docs/blob/main/integration-of-algebra-integral-protocol/contracts-api/Core/interfaces/pool/IAlgebraPoolState.md)

Applied to current code:

1.  `withdraw()` snapshots `fees0/fees1` before `_collect()` calls `burn(...)`.
2.  `burn(...)` updates fee accounting for the position.
3.  `collect(...)` then uses limits derived from the pre-burn snapshot.

This ordering is exactly why the fee component can be stale.

### Impact

This creates order-dependent payout and economic unfairness:

-   early withdrawers can receive less than their fair pro-rata share of accrued fees
-   remaining/later withdrawers can capture residual fees that should have been paid earlier
-   outcome depends on withdrawal ordering, not only share ownership
-   users may hit slippage checks unexpectedly because returned amounts are lower than expected

The protocol is not halted, but value can be redistributed between LP holders in a way that is exploitable by timing.