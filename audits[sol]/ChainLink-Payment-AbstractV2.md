reported at 2026/03/25

## L-01: CoW settlement path allows dust-level partial executions, creating order churn and operational overhead

**Severity**: Low

**Impacted Contracts**

-   \[GPV2CompatibleAuction.sol\]
-   \[BaseAuction.sol\]

## Summary

The direct bidding path enforces a minimum USD notional (`minBidUsdValue`) in \[`BaseAuction.bid()`\], but the CoW EIP-1271 path does not apply an equivalent dust floor in \[`GPV2CompatibleAuction.isValidSignature()`\].

In the CoW path, orders are required to be `partiallyFillable`, and settlement can execute very small partial amounts. This means dust-level executions can still pass protocol checks as long as the limit-price condition is respected.

## Root Cause

-   \[`BaseAuction.bid()`\] enforces `bidUsdValue >= s_minBidUsdValue`.
-   \[`GPV2CompatibleAuction.isValidSignature()`\] validates order identity, token pair, balance, expiry, and limit price, but does not enforce a minimum USD notional like `minBidUsdValue`.
-   The same function explicitly requires `order.partiallyFillable == true`, so very small partial executions are a valid CoW execution mode.

As a result, the system has asymmetric dust protection between the native bid path and the CoW settlement path.

## Impact

This issue is primarily an operational/liveness concern rather than a direct fund-loss vector:

-   tiny partial fills can increase order update/invalidation churn
-   off-chain relayer and workflow load can increase due to frequent order maintenance
-   more stale-quantity attempts from solvers may occur between workflow syncs

Economic protection is still partially preserved by the on-chain limit-price check (`InsufficientBuyAmount`), so this is rated **Low**.

## Recommended Mitigation

Apply a consistent anti-dust policy across execution paths:

1.  Add a CoW-path minimum order notional check in `isValidSignature()` (for example based on `order.sellAmount` USD value).
2.  Enforce off-chain relayer policy for minimum executable order size and avoid posting tiny replacement orders.
3.  If minimizing churn is more important than partial-fill flexibility, consider disabling partial fills for this integration path.

Note: a minimum on `order.sellAmount` reduces dust orders, but does not fully prevent tiny partial executions by itself. If full protection is required, pair on-chain checks with strict off-chain order-management rules.

## L-02: `PriceManager.transmit()` accepts out-of-order Data Streams reports and can roll back the stored price

**Severity**: Low

**Impacted Contracts**

-   \[PriceManager.sol\]
-   \[BaseAuction.sol\]
-   \[GPV2CompatibleAuction.sol\]

## Summary

`PriceManager.transmit()` verifies that a Data Streams report is signed and not older than the configured `stalenessThreshold`, but it does not enforce that `report.observationsTimestamp` is newer than the timestamp already stored for the same asset.

As a result, an older but still valid signed report can overwrite a newer stored report. Once this happens, the protocol continues using the rolled-back price until the stale-path fallback to the configured Chainlink Data Feed is triggered.

Because the cached price is consumed by auction start logic, bid validation, `getAssetOutAmount()`, and GPv2 order validation, this bug can misprice live auctions and weaken the auction curve invariant described in the contest README.

## Root Cause

In \[`PriceManager.transmit()`\], each verified report is written directly into `s_dataStreamsPrice[asset]`:

```php
if (report.observationsTimestamp < block.timestamp - feedInfo.stalenessThreshold) {
  revert Errors.StaleFeedData();
}

s_dataStreamsPrice[asset] =
  DataStreamsPriceInfo({usdPrice: usdPrice.toUint224(), timestamp: report.observationsTimestamp});
```

This only checks whether the report is too old relative to `block.timestamp`. It does not check whether the new report is older than the currently stored report for the same asset.

Therefore, the following sequence is accepted:

1.  Store report A with timestamp `T2` and price `P2`
2.  Later submit report B with timestamp `T1`, where `T1 < T2`
3.  As long as `T1` is still within `stalenessThreshold`, report B is accepted
4.  The protocol state rolls back from `(P2, T2)` to `(P1, T1)`

## Impact

This is not just a stale UI/view issue. The rolled-back Data Streams price is directly used by core auction paths:

-   \[`BaseAuction.bid()`\] uses `_getAssetPrice(asset, true)` to value the bid in USD and compute the required `assetOutAmount`
-   \[`BaseAuction.getAssetOutAmount()`\] uses the cached price to quote auction output
-   \[`BaseAuction.checkUpkeep()`\] uses asset prices to decide whether an auction should start or end
-   \[`GPV2CompatibleAuction.isValidSignature()`\] derives the minimum acceptable buy amount from the cached price

Practical consequences:

-   an auctioned asset can be undervalued or overvalued using an older report
-   `minBuyAmount` checks can be relaxed or tightened incorrectly
-   bids can settle against an outdated price curve
-   auction start/end eligibility can be evaluated from rolled-back prices

I rate this as **Low** because exploitability is constrained by trust assumptions:

-   `transmit()` is restricted to `PRICE_ADMIN_ROLE`
-   in the intended architecture, this role is exercised by the trusted workflow/forwarder pipeline
-   no permissionless user can directly inject arbitrary reports

So this is primarily a robustness/integrity issue under out-of-order delivery or replay within a trusted pipeline, rather than a broadly attacker-controlled loss vector.

## Proof of Concept

A runnable PoC was added in:

```perl
L-01: CoW settlement path allows dust-level partial executions, creating order churn and operational overheadfunction testSubmissionValidity() public {
    uint256 initialTimestamp = block.timestamp;
    uint256 newerTimestamp = initialTimestamp + 30 minutes;
    uint256 olderTimestamp = initialTimestamp + 10 minutes;

    // First store a fresher report.
    vm.warp(newerTimestamp);
    _transmitPrices(4_500e18, 1e18, 20e18);

    (uint256 latestPrice, uint256 latestUpdatedAt, bool latestIsValid) = auction.getAssetPrice(address(mockWETH));
    assertEq(latestPrice, 4_500e18, "newer report should set the latest WETH price");
    assertEq(latestUpdatedAt, newerTimestamp, "newer report should set the latest timestamp");
    assertTrue(latestIsValid, "newer report should be valid");

    // Then submit an older but still non-stale signed report. transmit() accepts it and rolls back state.
    _transmitPricesWithObservationTimestamp(3_000e18, 1e18, 20e18, uint32(olderTimestamp));

    (uint256 rolledBackPrice, uint256 rolledBackUpdatedAt, bool rolledBackIsValid) =
      auction.getAssetPrice(address(mockWETH));

    assertEq(rolledBackPrice, 3_000e18, "older report should overwrite the fresher stored price");
    assertEq(rolledBackUpdatedAt, olderTimestamp, "timestamp should roll back to the older observation");
    assertTrue(rolledBackIsValid, "rolled back report remains valid until staleness fallback triggers");
    assertLt(rolledBackUpdatedAt, latestUpdatedAt, "PoC requires the stored timestamp to move backwards");
  }
```

The PoC demonstrates:

1.  a newer WETH report is stored first
2.  an older but still non-stale report is submitted afterwards
3.  `auction.getAssetPrice(mockWETH)` returns the older price and older timestamp

## Recommended Mitigation

Reject any report whose `observationsTimestamp` is not strictly newer than the currently stored timestamp:

```php
uint32 lastTimestamp = s_dataStreamsPrice[asset].timestamp;
if (report.observationsTimestamp <= lastTimestamp) {
  continue; // ignore stale or duplicate reports
}

s_dataStreamsPrice[asset] =
  DataStreamsPriceInfo({usdPrice: usdPrice.toUint224(), timestamp: report.observationsTimestamp});
```

This preserves monotonic price updates per asset and prevents rollback from out-of-order or replayed reports.