## [M-1][duplicated] Permissionless `rebalance()` lets anyone force full-vault repositioning

### Description

`AlphaProVault.rebalance()` is meant to be a manager or keeper action, but the current code only checks the caller when `rebalanceDelegate` is non-zero. If `rebalanceDelegate == address(0)`, the auth check is skipped and anyone can call `rebalance()`.

That matters because `rebalance()` does not just update a timestamp. It withdraws all current Uniswap V3 liquidity, reads the current spot tick from `pool.slot0()`, then redeploys the vault around that spot tick. A public caller can therefore choose the timing of a full-vault reposition.

On a mainnet fork, this turns an otherwise losing sandwich path into a profitable one. With a vault funded with about `33.6M USDC` notional value, the unauthorized rebalance adds `899,596.376722 USDC` of value to the attacker path, about `2.68%` of vault TVL in the tested scenario.

### Severity

High.

The bug is a real access-control bypass, and the fork PoC shows economic extraction, not only state mutation. The impact scales with vault TVL, pool depth, and the rebalance parameters.

### Affected Code

-   `contracts/AlphaProVault.sol:374-381`: `rebalance()` skips auth when `rebalanceDelegate == address(0)`.
-   `contracts/AlphaProVault.sol:144-178`: `initialize()` accepts a zero `rebalanceDelegate`.
-   `contracts/AlphaProVault.sol:806-809`: `setRebalanceDelegate(address(0))` is allowed.
-   `contracts/AlphaProVault.sol:395-450`: `rebalance()` withdraws all liquidity and redeploys it using the current spot tick.

Core issue:

```scss
function rebalance() external override nonReentrant {
    checkCanRebalance();
    if (rebalanceDelegate != address(0)) {
        require(
            msg.sender == manager || msg.sender == rebalanceDelegate,
            "rebalanceDelegate"
        );
    }

    // withdraw all liquidity, read pool.slot0(), redeploy positions
}
```

When the delegate is zero, there is no `msg.sender` check at all.

### Attack Path

1.  A vault has `rebalanceDelegate == address(0)`, or the live deployment has no delegate gate.
2.  The attacker moves the pool spot price and keeps it inside the configured TWAP bound.
3.  The attacker calls `rebalance()` directly.
4.  The vault removes all liquidity and redeploys around the attacker-influenced spot tick.
5.  The attacker reverses the price move and extracts value from the vault's new position.

`checkCanRebalance()` only checks timing, tick movement, TWAP deviation, and tick boundaries. It does not authenticate the caller.

### Mainnet State

Repository deployment addresses:

```cpp
AlphaProVaultFactory = 0x8C554F200B1EEECdE99370Fe6284B15d23E50E07
AlphaProVault template = 0x8cBC885Ece6C515e092BA1b3E24b567a7d91572c
```

At block `25164013`, the factory had `26` vaults. The live vault implementation did not expose `rebalanceDelegate()`:

```bash
RPC=<mainnet_rpc_url>

cast call 0x28C5e6AD659aCb0007A998A8B5d32E7910419757 \
  'rebalanceDelegate()(address)' \
  --rpc-url "$RPC" || echo "NO_REBALANCE_DELEGATE_FUNCTION"
```

Observed result:

```
NO_REBALANCE_DELEGATE_FUNCTION
```

So the live deployment is not just a zero-delegate case. It is an older version where `rebalance()` has no delegate-based authorization gate.

I used this command to check the live rebalance parameters:

```bash
SummaryRPC=<mainnet_rpc_url>
FACTORY=0x8C554F200B1EEECdE99370Fe6284B15d23E50E07
N=$(cast call "$FACTORY" 'numVaults()(uint256)' --rpc-url "$RPC")

for i in $(seq 0 $((N-1))); do
  V=$(cast call "$FACTORY" 'vaults(uint256)(address)' "$i" --rpc-url "$RPC")
  NAME=$(cast call "$V" 'name()(string)' --rpc-url "$RPC" 2>/dev/null || printf 'ERR')
  MAX=$(cast call "$V" 'maxTwapDeviation()(int24)' --rpc-url "$RPC" 2>/dev/null || printf 'ERR')
  DUR=$(cast call "$V" 'twapDuration()(uint32)' --rpc-url "$RPC" 2>/dev/null || printf 'ERR')
  SHOULD=$(cast call "$V" 'shouldRebalance()(bool)' --rpc-url "$RPC" 2>/dev/null || printf 'ERR')
  printf '%s %s %s maxTwapDeviation=%s twapDuration=%s shouldRebalance=%s\n' \
    "$i" "$V" "$NAME" "$MAX" "$DUR" "$SHOULD"
done
```

Result summary at block `25164013`:

-   `26 / 26` vaults had no `rebalanceDelegate()` function in the deployed implementation.
-   `25 / 26` vaults used `maxTwapDeviation = 100` and `twapDuration = 60`.
-   `1 / 26` vault used `maxTwapDeviation = 300` and `twapDuration = 60`.
-   `16 / 26` vaults returned `shouldRebalance() == true`.

### Recommended Fix

Always authenticate `rebalance()`.

```scss
function rebalance() external override nonReentrant {
    require(
        msg.sender == manager || msg.sender == rebalanceDelegate,
        "rebalance auth"
    );

    checkCanRebalance();
    ...
}
```

If `address(0)` is meant to mean "no delegate", then only the manager should be allowed:

```scss
function rebalance() external override nonReentrant {
    if (rebalanceDelegate == address(0)) {
        require(msg.sender == manager, "rebalance auth");
    } else {
        require(
            msg.sender == manager || msg.sender == rebalanceDelegate,
            "rebalance auth"
        );
    }

    checkCanRebalance();
    ...
}
```

If public keeper rebalancing is intended, it should be explicit and documented, and the rebalance logic needs stronger MEV protection before full-vault repositioning is exposed to arbitrary callers.

### Proof of Concept

-   `test/poc-f01-economic-fork.test.ts`
-   `hardhat.fork.poc.config.ts`

Run:

```bash
MAINNET_RPC_URL=<mainnet_rpc_url> \
  npx hardhat --config hardhat.fork.poc.config.ts test test/poc-f01-economic-fork.test.ts
```

The fork test uses the real Uniswap V3 WETH/USDC 0.3% pool and a vault with:

-   `rebalanceDelegate = address(0)`
-   `maxTwapDeviation = 100`
-   `twapDuration = 60`
-   `16,800,000 USDC` and `8,000 WETH`

Output:

```sql
without forced rebalance value delta USDC: -253297.425012
with forced rebalance value delta USDC: 646298.95171
incremental value extracted from forced rebalance USDC: 899596.376722
```

The same two-swap path loses money without the forced rebalance, but becomes profitable when the attacker can insert `rebalance()` between the swaps. The unauthorized rebalance contributes `899,596.376722 USDC` of incremental value in this run.

#### Full PoC code(ignore)
...