## [L-1]: Inconsistent token width can duplicate supplied-asset accounting

`supplyToBlp(uint64 token, ...)` and `addMultisigSupplyToken(uint64 spotToken)` accept 64-bit token ids, but whitelist checks and backing reads cast them to `uint32`. Different `uint64` values with the same low 32 bits can therefore be registered as distinct supplied entries while being valued as the same whitelisted token, potentially double-counting backing in `totalBackingSigned()`. This is Operator-gated, but should be normalized defensively.

> eg: token = (highBits << 32) | whitelistedSpotToken

```go
// MonetrixVault.supplyToBlp(uint64 token, uint64 l1Amount) external onlyOperator whenOperatorNotPaused {
  ...
  if (token != uint64(HyperCoreConstants.USDC_TOKEN_INDEX)) {
@>  require(config.isSpotWhitelisted(uint32(token)), "spot not whitelisted");
    perpIndex = config.spotToPerp(uint32(token));
  }
  MonetrixAccountant(accountant).notifyVaultSupply(token, perpIndex);
}
```

```php
// MonetrixAccountant.sol
function notifyVaultSupply(uint64 spotToken, uint32 perpIndex) external onlyVault {
  if (!_vaultSupplyKnown[spotToken]) {
    _vaultSupplyKnown[spotToken] = true;
    vaultSupplied.push(SuppliedAsset({spotToken: spotToken, perpIndex: perpIndex}));
  }
}

function _readL1Backing(address account, SuppliedAsset[] storage suppliedList)
  internal
  view
  returns (int256 total)
{
  ...
  total += int256(
@>  PrecompileReader.suppliedNotionalUsdcFromPerp(uint32(a.spotToken), a.perpIndex, account)
  );
}
```

Recommendation: require supplied token ids to fit in `uint32`, or store `SuppliedAsset.spotToken` as `uint32` consistently.

## [L-2]: Time-based windows continue elapsing while the protocol is paused

The protocol's pause switches block selected operations, but they do not freeze or extend any time-based windows. Existing redemption requests, sUSDM unstake requests, bridge intervals, and settlement elapsed time all continue to use raw `block.timestamp` while the relevant contract is paused.

This means that, after an unpause, users or operators may be able to immediately execute actions whose waiting period elapsed entirely during the pause. In particular, `settleDailyPnL()` also includes the paused duration in the APR-cap `elapsed` calculation, increasing the maximum post-unpause yield proposal. This is still bounded by distributable surplus and available EVM USDC, so the issue is mainly a semantics/documentation and operational-risk concern.

```javascript
// MonetrixVault.sol
function requestRedeem(uint256 usdmAmount) external ... whenNotPaused ... {
  ...
@> cooldownEnd: SafeCast.toUint64(block.timestamp + config.redeemCooldown()),
  ...
}

function claimRedeem(uint256 requestId) external nonReentrant whenNotPaused requireWired {
  RedeemRequest memory req = redeemRequests[requestId];
@> require(req.usdmAmount > 0 && msg.sender == req.owner && block.timestamp >= req.cooldownEnd, "invalid claim");
  ...
}
```

```scss
// sUSDM.sol
function cooldownShares(uint256 shares) external nonReentrant whenNotPaused returns (uint256 requestId) {
  ...
@> cooldownEnd: block.timestamp + config.unstakeCooldown()
  ...
}

function claimUnstake(uint256 requestId) external nonReentrant whenNotPaused {
  ...
@> if (block.timestamp < req.cooldownEnd) revert CooldownNotExpired(requestId, req.cooldownEnd);
  ...
}
```

```java
// MonetrixAccountant.sol
function settleDailyPnL(uint256 proposedYield) external onlyVault returns (uint256 distributable) {
@> require(block.timestamp >= lastSettlementTime + minSettlementInterval, "Accountant: settlement too early");
  ...
@> uint256 elapsed = block.timestamp - lastSettlementTime;
  uint256 cap =
    (usdm.totalSupply() * IMonetrixConfigReader(config).maxAnnualYieldBps() * elapsed) / (10_000 * 365 days);
  ...
}
```

Recommendation: explicitly document that pause does not stop protocol time. If paused time is intended to be excluded, track cumulative paused duration per relevant module and subtract it from cooldown/interval/elapsed calculations, or extend pending deadlines when unpausing.

## [L-3]: valid same-block bridge calls can desynchronize principal accounting because HyperCore precompile reads are block-start snapshots

HyperCore writes are not atomic with the HyperEVM transaction that enqueues them. The official Hyperliquid docs state that read precompile values "match the latest HyperCore state at the time the EVM block is constructed", while `CoreWriter` at `0x3333333333333333333333333333333333333333` emits a log to be processed later as a HyperCore action. The official interaction-timing page further specifies that, in a block producing a HyperEVM block, the order is: EVM block is built, then EVM->Core transfers are processed, then CoreWriter actions are processed.

References:

-   [Hyperliquid docs: Interacting with HyperCore](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore)
-   [Hyperliquid docs: Interaction timings](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interaction-timings)

`bridgePrincipalFromL1()` decrements `outstandingL1Principal` before `_sendL1Bridge()` enqueues the L1 `sendAsset` action. `_sendL1Bridge()` checks the Vault's L1 USDC availability through precompiles, but that check is based on the HyperCore snapshot from the start of the EVM block. If multiple principal bridge calls are included in the same HyperEVM block, each call can validate against the same stale L1 balance even though the queued CoreWriter actions will later consume that balance sequentially.

```python
[QA-1]: Inconsistent token width can duplicate supplied-asset accounting// MonetrixVault.sol
function bridgePrincipalFromL1(uint256 amount) external onlyOperator requireWired whenOperatorNotPaused {
  require(amount > 0 && amount <= redemptionShortfall() && amount <= outstandingL1Principal, "invalid bridge amount");
@> outstandingL1Principal -= amount;
  _sendL1Bridge(amount);
  emit PrincipalBridgedFromL1(amount);
}

function _sendL1Bridge(uint256 amount) internal {
  uint64 usdcToken = uint64(HyperCoreConstants.USDC_TOKEN_INDEX);
@> uint256 l1Available = uint256(PrecompileReader.spotBalance(address(this), usdcToken).total);
  if (pmEnabled) {
@>   l1Available += uint256(PrecompileReader.suppliedBalance(address(this), usdcToken));
  }
  require(
    l1Available >= TokenMath.usdcEvmToL1Wei(amount), "L1 USDC insufficient (unwind hedge or wait for settlement)"
  );
@> ActionEncoder.sendBridgeToL1(amount);
}
```

Example:

-   Vault has `100,000 USDC` of `outstandingL1Principal`.
-   Only `50,000 USDC` is currently available as L1 spot/supplied USDC.
-   Redemption shortfall is at least `80,000 USDC`.
-   The Operator submits two same-block calls: `bridgePrincipalFromL1(40,000e6)` and `bridgePrincipalFromL1(40,000e6)`.

Both EVM calls can pass because both read the same block-start L1 balance of `50,000 USDC`. The Vault immediately decreases `outstandingL1Principal` by `80,000 USDC`. Later, HyperCore executes the queued `sendAsset` actions sequentially: the first action succeeds and leaves only `10,000 USDC`, while the second action can fail due to insufficient L1 spot/supplied balance. The failed CoreWriter action does not roll back the already-committed HyperEVM state.

This does not directly create false backing, because `MonetrixAccountant.totalBackingSigned()` reads actual EVM balances and HyperCore precompile state rather than `outstandingL1Principal`. However, it can leave `outstandingL1Principal` lower than the amount of principal that was actually bridged back. That can reduce or block future principal bridge calls and delay redemption funding until manual operational recovery.

This is categorized as QA because the affected functions are Operator/Governor gated and the README explicitly treats the Operator as a trusted hot-wallet role. Still, the current accounting assumes successful asynchronous CoreWriter execution and does not track pending or failed L1->EVM bridge actions.

Recommendation: track pending L1->EVM bridge amounts separately and only decrement `outstandingL1Principal` after the corresponding USDC is observed on the EVM side or used to fund redemptions. At minimum, add a same-block pending bridge accumulator to ensure cumulative bridge requests in one EVM block cannot exceed the precompile snapshot balance, or restrict L1->EVM principal bridge calls to one per EVM block.

## [L-5]: sUSDM reports 6 decimals even though its ERC-4626 share accounting uses a 6-decimal offset

The official ERC-4626 specification requires tokenized vault shares to implement the ERC-20 optional metadata extension and states that ERC-20 operations such as `balanceOf`, `transfer`, and `totalSupply` operate on the vault shares. The same specification also recommends that vault implementers avoid confusing `decimals` metadata where possible because front-ends and other off-chain users still commonly rely on it.

References:

-   [EIP-4626: Tokenized Vaults](https://eips.ethereum.org/EIPS/eip-4626)
-   [OpenZeppelin Contracts v5 ERC4626 API](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC4626)

OpenZeppelin's ERC4626 implementation uses `_decimalsOffset()` as the decimal offset between the underlying asset's decimals and the vault share decimals. Its documented `decimals()` behavior is to add the offset on top of the underlying asset decimals. The local OpenZeppelin implementation used by this repository follows the same rule:

```csharp
// ERC4626Upgradeable.sol
function decimals() public view virtual override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {
    ERC4626Storage storage $ = _getERC4626Storage();
@>  return $._underlyingDecimals + _decimalsOffset();
}

function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
@>  return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
}
```

`USDM` is a 6-decimal asset:

```csharp
// USDM.sol
function decimals() public pure override returns (uint8) {
@>  return 6;
}
```

However, `sUSDM` overrides `_decimalsOffset()` to `6` while also overriding `decimals()` back to `6`:

```csharp
// sUSDM.sol
function _decimalsOffset() internal pure override returns (uint8) {
@>  return 6;
}

function decimals() public pure override returns (uint8) {
@>  return 6;
}
```

This creates a metadata mismatch. The actual ERC-4626 share math mints `1e12` raw shares for a `1e6` raw USDM deposit when the vault is empty, because the configured offset is `6`. The project's own tests also use `1e12` as one whole sUSDM share unit. But because `sUSDM.decimals()` reports `6`, wallets, explorers, indexers, and other ERC-20/ ERC-4626 integrations that format balances using token metadata will display `1e12` raw shares as `1,000,000 sUSDM` instead of `1 sUSDM`.

This does not directly break the on-chain ERC-4626 arithmetic because conversions use raw balances and `_decimalsOffset()`. The issue is an integration and UX correctness risk: external systems can overstate user balances, total supply, or share-denominated quotes by `1e6` when they rely on `decimals()` for human-readable formatting.

Recommendation: either remove the `decimals()` override so OpenZeppelin reports `asset.decimals() + _decimalsOffset()` (`12` for a 6-decimal USDM asset with offset `6`), or set `_decimalsOffset()` to `0` if the intended public sUSDM share precision is actually 6 decimals. The first option matches the existing conversion math and current tests that treat `1e12` raw shares as one whole sUSDM.