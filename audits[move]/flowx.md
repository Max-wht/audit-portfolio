reported at 2026/05/25

## [Info-1] pool::swap() uses the exact-in flag to select protocol fee rate, causing protocol and LP fee misallocation

### Summary

`pool::swap()` picks the protocol fee denominator from `exact_in`. That value only says whether the user specified an input amount or an output amount; it does not say which token is being paid into the pool.

Protocol fees are charged on the input token side, so the denominator needs to follow the swap direction (`x_for_y`). If X and Y protocol fee rates are ever configured differently, two normal swap paths will split fees with the wrong rate.

This does not make the swapper pay more or less total swap fee. The issue is in the internal split: some fee that should go to LPs can go to the protocol treasury instead, or vice versa. Based on the current mainnet data I checked, the deployed pools are not currently in the unequal-rate configuration needed to trigger the issue.

### Affected Components

-   `sources/pool.move`
    -   `pool::swap()` protocol fee rate selection at lines 476-480
    -   protocol fee accounting at lines 558-668
    -   `set_protocol_fee_rate()` encoding at lines 1004-1024
-   `sources/swap_router.move`
    -   `swap_exact_y_to_x()` at lines 50-66
    -   `swap_x_to_exact_y()` at lines 127-152
    -   high-level `swap_exact_input()` / `swap_exact_output()` routes

### Vulnerability Details

The pool supports two separate protocol fee denominators, one for token X and one for token Y:

move Copy

```move
self.protocol_fee_rate = protocol_fee_rate_x + (protocol_fee_rate_y << 4);
```

So the lower nibble is the X-side denominator, and the upper nibble is the Y-side denominator.

The swap path is where the selector changes meaning:

```move
let protocol_fee_rate = if (exact_in) {
    self.protocol_fee_rate % 16
} else {
    self.protocol_fee_rate >> 4
};
```

That branch is not selecting by token. It is selecting by swap mode.

The input token is already represented by `x_for_y`:

-   `x_for_y == true`: token X is paid into the pool and token Y is taken out.
-   `x_for_y == false`: token Y is paid into the pool and token X is taken out.

The accounting at the end of the swap also follows `x_for_y`:

```move
if (x_for_y) {
    self.fee_growth_global_x = state.fee_growth_global;
    self.protocol_fee_x = self.protocol_fee_x + state.protocol_fee;
} else {
    self.fee_growth_global_y = state.fee_growth_global;
    self.protocol_fee_y = self.protocol_fee_y + state.protocol_fee;
};
```

So the function adds protocol fees to the X bucket when X is the input side, and to the Y bucket when Y is the input side. The denominator used to split that fee should be chosen the same way.

### Incorrect Swap Modes

|   Swap path   | `x_for_y` | `exact_in` | Input token | Current denominator | Correct denominator |
|---------------|-------|----------|-------------|---------------------|---------------------|
| `swap_exact_x_to_y` | true  |   true   |      X      |          X          |          X          |
| `swap_exact_y_to_x` | false |   true   |      Y      |          X          |          Y          |
| `swap_x_to_exact_y` | true  |  false   |      X      |          Y          |          X          |
| `swap_y_to_exact_x` | false |  false   |      Y      |          Y          |          Y          |

This means 2 of the 4 swap modes are wrong:

-   `swap_exact_y_to_x()` passes `pool::swap(pool, false, true, ...)`, so Y is the input token but the X fee denominator is used.
-   `swap_x_to_exact_y()` passes `pool::swap(pool, true, false, ...)`, so X is the input token but the Y fee denominator is used.

Both are normal public swap paths. They are also reachable through the high-level router functions, so this is not limited to an internal-only call pattern.

### Impact

The condition needed for impact is simple: `protocol_fee_rate_x` and `protocol_fee_rate_y` must be different.

Example:

-   `protocol_fee_rate_x = 4`
-   `protocol_fee_rate_y = 10`
-   A `Y -> X` exact-input swap generates `100 Y` of swap fee.

Expected accounting:

```text
protocol fee = 100 / 10 = 10 Y
LP fee       = 90 Y
```

Accounting with the current code:

```text
protocol fee = 100 / 4 = 25 Y
LP fee       = 75 Y
```

In that example, the protocol takes `15 Y` more than intended, and LPs receive `15 Y` less. If the two denominators are reversed, the direction of the error reverses as well: the protocol under-collects and LPs receive too much.

I would not describe this as price manipulation or a trader payment bypass. The trader pays the same total fee. The concrete impact is incorrect fee attribution between the protocol and LPs, which can accumulate over trading volume once unequal X/Y protocol fee rates are used.

### Current Mainnet Status

I also checked the current mainnet state so the report does not overstate production impact:

```text
package: 0xe882cd54551e73e64ff5b257146a0c5264546974cf00d78ecc871017cb22df67
module:  0x25929e7f29e0a30eb4e692952ba1b5b65a3a4d65ab5f2a32e1ba3edcb587f26d::pool
source:  Sui mainnet fullnode RPC
```

The result was:

|                           Item                           | Count |
|----------------------------------------------------------|-------|
|                    `PoolCreated` events                    | 1189  |
|                Current pool objects read                 | 1189  |
|           Pools with non-zero `protocol_fee_rate`            |   3   |
| Current pools where `protocol_fee_rate_x != protocol_fee_rate_y` |   0   |
|           `SetProtocolFeeRate` historical events           |   3   |
|   Historical events setting unequal X/Y protocol rates   |   0   |

What this means:

-   I did not find evidence that the bug has already caused wrong fee accounting on mainnet.
-   The current production configuration avoids the trigger condition because the observed non-zero protocol fee pools use equal X/Y rates.
-   The code still allows unequal X/Y protocol fee rates through `set_protocol_fee_rate()`. A future configuration change can therefore activate the bug without changing the deployed swap code.

Validation steps

### Proof of Concept

The code path can be followed directly:

1.  Configure a pool with unequal protocol fee denominators:

```move
pool::set_protocol_fee_rate(&admin_cap, &mut pool, 4, 10, &versioned, &ctx);
```

This stores:

```text
self.protocol_fee_rate = 4 + (10 << 4)
```

2.  Execute an exact-input Y-to-X swap:


```move
pool::swap(&mut pool, false, true, amount_y_in, sqrt_price_limit, &versioned, &clock, &ctx);
```

3.  In `pool::swap()`, the current code enters the `exact_in == true` branch and selects:


```move
self.protocol_fee_rate % 16
```

This returns `4`, which is the X-token denominator.

4.  Because `x_for_y == false`, the same swap later accounts the protocol fee as Y-token fees:

```move
self.protocol_fee_y = self.protocol_fee_y + state.protocol_fee;
```

So this Y-denominated protocol fee is split with the X-side setting.

The same mismatch happens in the other direction for exact-output X-to-Y swaps:

```move
pool::swap(&mut pool, true, false, amount_y_out, sqrt_price_limit, &versioned, &clock, &ctx);
```

Here X is the input token, but `exact_in == false` selects the Y-side denominator.

I would use the following regression test as the clearest proof. It belongs in the existing `#[test_only] module flowx_clmm::test_pool` in `sources/pool.move`.

The test creates three otherwise identical pools:

-   one pool with the mixed configuration `(4, 10)`
-   one control pool with `(4, 4)`
-   one control pool with `(10, 10)`

For an exact Y-to-X swap, the mixed pool should behave like the `(10, 10)` control, because Y is the input token. With the current code, it behaves like the `(4, 4)` control instead. That shows the X denominator is being used for a Y-input swap.

```move
#[test]
fun test_protocol_fee_rate_selector_uses_wrong_denominator_for_y_input() {
    let ctx = tx_context::dummy();
    let clock = clock::create_for_testing(&mut ctx);
    let versioned = versioned::create_for_testing(&mut ctx);
    let admin_cap = flowx_clmm::admin_cap::create_for_testing(&mut ctx);

    clock::set_for_testing(&mut clock, 10000);
    let (fee_rate, tick_spacing) = (3000, 60);
    let (min_tick, max_tick) = (
        test_utils::get_min_tick(tick_spacing),
        test_utils::get_max_tick(tick_spacing)
    );

    let bug_pool = pool::create_for_testing<SUI, USDC>(fee_rate, tick_spacing, &mut ctx);
    let control_x_pool = pool::create_for_testing<SUI, USDC>(fee_rate, tick_spacing, &mut ctx);
    let control_y_pool = pool::create_for_testing<SUI, USDC>(fee_rate, tick_spacing, &mut ctx);

    pool::initialize_for_testing(&mut bug_pool, test_utils::encode_sqrt_price(1, 1), &clock, &ctx);
    pool::initialize_for_testing(&mut control_x_pool, test_utils::encode_sqrt_price(1, 1), &clock, &ctx);
    pool::initialize_for_testing(&mut control_y_pool, test_utils::encode_sqrt_price(1, 1), &clock, &ctx);

    // bug_pool should use Y denominator 10 for a Y-input swap.
    pool::set_protocol_fee_rate(&admin_cap, &mut bug_pool, 4, 10, &versioned, &ctx);

    // Control pools isolate which denominator is actually used.
    pool::set_protocol_fee_rate(&admin_cap, &mut control_x_pool, 4, 4, &versioned, &ctx);
    pool::set_protocol_fee_rate(&admin_cap, &mut control_y_pool, 10, 10, &versioned, &ctx);

    let position_bug = position::create_for_testing(
        pool::pool_id(&bug_pool), fee_rate, pool::coin_type_x(&bug_pool), pool::coin_type_y(&bug_pool),
        min_tick, max_tick, &mut ctx
    );
    pool::modify_liquidity<SUI, USDC>(
        &mut bug_pool, &mut position_bug, i128::from(2000000000),
        balance::create_for_testing(2000000000), balance::create_for_testing(2000000000),
        &versioned, &clock, &ctx
    );
    position::destroy_for_testing(position_bug);

    let position_x = position::create_for_testing(
        pool::pool_id(&control_x_pool), fee_rate, pool::coin_type_x(&control_x_pool), pool::coin_type_y(&control_x_pool),
        min_tick, max_tick, &mut ctx
    );
    pool::modify_liquidity<SUI, USDC>(
        &mut control_x_pool, &mut position_x, i128::from(2000000000),
        balance::create_for_testing(2000000000), balance::create_for_testing(2000000000),
        &versioned, &clock, &ctx
    );
    position::destroy_for_testing(position_x);

    let position_y = position::create_for_testing(
        pool::pool_id(&control_y_pool), fee_rate, pool::coin_type_x(&control_y_pool), pool::coin_type_y(&control_y_pool),
        min_tick, max_tick, &mut ctx
    );
    pool::modify_liquidity<SUI, USDC>(
        &mut control_y_pool, &mut position_y, i128::from(2000000000),
        balance::create_for_testing(2000000000), balance::create_for_testing(2000000000),
        &versioned, &clock, &ctx
    );
    position::destroy_for_testing(position_y);

    // exact Y -> X: x_for_y=false, exact_in=true.
    let (x_out, y_out, receipt) = pool::swap(
        &mut bug_pool, false, true, test_utils::expand_to_9_decimals(1),
        tick_math::max_sqrt_price() - 1, &versioned, &clock, &ctx
    );
    let (amount_x_debt, amount_y_debt) = pool::swap_receipt_debts(&receipt);
    pool::pay(&mut bug_pool, receipt, balance::create_for_testing(amount_x_debt), balance::create_for_testing(amount_y_debt), &versioned, &ctx);
    balance::destroy_for_testing(x_out);
    balance::destroy_for_testing(y_out);

    let (x_out, y_out, receipt) = pool::swap(
        &mut control_x_pool, false, true, test_utils::expand_to_9_decimals(1),
        tick_math::max_sqrt_price() - 1, &versioned, &clock, &ctx
    );
    let (amount_x_debt, amount_y_debt) = pool::swap_receipt_debts(&receipt);
    pool::pay(&mut control_x_pool, receipt, balance::create_for_testing(amount_x_debt), balance::create_for_testing(amount_y_debt), &versioned, &ctx);
    balance::destroy_for_testing(x_out);
    balance::destroy_for_testing(y_out);

    let (x_out, y_out, receipt) = pool::swap(
        &mut control_y_pool, false, true, test_utils::expand_to_9_decimals(1),
        tick_math::max_sqrt_price() - 1, &versioned, &clock, &ctx
    );
    let (amount_x_debt, amount_y_debt) = pool::swap_receipt_debts(&receipt);
    pool::pay(&mut control_y_pool, receipt, balance::create_for_testing(amount_x_debt), balance::create_for_testing(amount_y_debt), &versioned, &ctx);
    balance::destroy_for_testing(x_out);
    balance::destroy_for_testing(y_out);

    // Bug behavior: (4,10) matches the X-denominator control, not the Y-denominator control.
    assert!(pool::protocol_fee_y(&bug_pool) == pool::protocol_fee_y(&control_x_pool), 0);
    assert!(pool::protocol_fee_y(&bug_pool) != pool::protocol_fee_y(&control_y_pool), 1);

    pool::destroy_for_testing(bug_pool);
    pool::destroy_for_testing(control_x_pool);
    pool::destroy_for_testing(control_y_pool);
    clock::destroy_for_testing(clock);
    versioned::destroy_for_testing(versioned);
    flowx_clmm::admin_cap::destroy_for_testing(admin_cap);
}
```

Expected result on the current implementation:

text Copy

```text
The test passes because bug_pool(4,10) behaves like control_x_pool(4,4)
for an exact Y -> X swap.
```

Expected result after the fix:

text Copy

```text
The first assertion should fail, and bug_pool(4,10) should instead match
control_y_pool(10,10) for the Y-denominated protocol fee.
```

### Test Coverage Gap

The current tests are easy to miss because they mostly exercise the right swap mechanics, but not the asymmetric fee configuration that exposes this bug. The protocol-fee-enabled swap tests use equal X/Y denominators, for example:

move Copy

```move
pool::set_protocol_fee_rate(&admin_cap, &mut pool, 6, 6, &versioned, &ctx);
```

When both denominators are equal, the wrong selector produces the same number as the right selector. That hides the bug.