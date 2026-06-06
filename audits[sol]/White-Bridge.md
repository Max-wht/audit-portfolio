## [Info-1] Source-chain deposits can be finalized without target-chain liquidity availability

### Summary

The bridge finalizes the source-chain side of a transfer before the target-chain side is proven to have enough liquidity to complete the withdrawal.

In `Bridge.bridgeTokens()`, the contract only validates the source-chain mapping and relayer signature. After that, it transfers tokens from the user into the source-chain bridge, accepts native coin deposits, or burns the received tokens when the source mapping uses `DepositType.Burn`.

```php
IMapper.MapInfo memory _mapInfo = _getMapInfo({mapId: bridgeTokensParams.bridgeParams.mapId});

require(_mapInfo.isAllowed, "Bridge: IsAllowed must be true");
require(_mapInfo.withdrawType == IMapper.WithdrawType.None, "Bridge: WithdrawType must be equal to None");
```

The source-chain deposit flow does not check whether the target-chain `Bridge` can later fulfill the matching withdrawal. This matters for target routes that use `WithdrawType.Unlock`, because those withdrawals require prefunded liquidity in the target-chain bridge.

For native coin withdrawals, `receiveTokens()` checks the target bridge's available coin balance only on the target chain:

```java
uint256 _contractBalance = address(this).balance - gasAccumulated;
require(
    _contractBalance >= receiveTokensParams.amount,
    "Bridge: Contract coins balance must be greater or equal amount"
);
```

For ERC20 `Unlock` withdrawals, the target-side transfer will revert if the target bridge does not hold enough token balance. In both cases, the failure happens after the user's source-chain assets have already been locked or burned. `WithdrawType.Mint` is less affected because it does not rely on prefunded bridge liquidity, assuming the target bridge has the required minting permission on the target token.

The relayer signature can act as an off-chain pre-check, but the signed message does not commit to target-chain liquidity or reserve capacity. If the relayer/backend signs without checking target liquidity, or if target liquidity is consumed after signing and before `receiveTokens()`, the source-chain transaction can still succeed while the target-chain withdrawal later fails.

### POC

**Validation steps:**
1. A user requests a bridge transfer.

2. The relayer signs the bridgeTokens() request. The signed fields include the user, recipient, target token, amount, gas amount, chain IDs, deadline, and salt, but do not include any target-chain liquidity commitment.

3. The user calls bridgeTokens() on the source chain. The function validates only the source-chain mapping and relayer signature, then locks or burns the user's source-chain assets and emits Deposit.

4. The relayer later calls receiveTokens() on the target chain.

5. If the target bridge has insufficient native coin or ERC20 liquidity, the target withdrawal reverts. The user has no permissionless refund or cancel path on the source chain.

This is a Low severity issue because bridgeTokens() requires a valid relayer signature, so the risk depends on relayer/backend liquidity checks, stale target-chain state, or operational misconfiguration. However, the on-chain source deposit still finalizes user funds without a target-side liquidity guarantee.

