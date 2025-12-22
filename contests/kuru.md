# KURU
[Contest Details](https://cantina.xyz/code/cdce21ba-b787-4df4-9c56-b31d085388e7)

## [H-01] Price-dependent execution compares against bid only, even for buys

### Summary
The KuruForwarder.executePriceDependent() function incorrectly uses only the bid price for all price-dependent order triggers, regardless of whether the order is a buy or sell. This causes buy orders to execute based on bid prices instead of ask prices, potentially leading to executions at worse-than-intended prices when spreads are wide. The severity is Medium due to silent value leakage during normal trading operations.

### Finding Description
The executePriceDependent() function in KuruForwarder.sol implements price-conditional meta-transaction execution but contains a fundamental flaw in its price checking logic. The function retrieves both bid and ask prices from the OrderBook's bestBidAsk() function but only uses the bid price for all comparisons.

The price condition check compares the user's breakpoint price against _currentBidPrice regardless of the order direction specified in req.selector. This means buy orders, which should execute against ask prices (the price users will pay), are incorrectly evaluated against bid prices (the price users would receive when selling).

The PriceDependentRequest struct contains a selector field that identifies the target function (buy vs sell), but this information is not used in the price validation logic. The function simply forwards the call after the price check without considering the order side.

### Impact Explanation
Users intending to execute buy orders when the ask price reaches a favorable level may experience unexpected executions when only the bid price meets their criteria while the ask price remains unfavorable. This is particularly problematic in markets with wide bid-ask spreads, where the bid and ask prices can differ significantly. Users receive worse execution prices than intended, resulting in silent value leakage during normal trading operations.

### Likelihood Explanation
This issue affects any price-dependent buy order execution when bid-ask spreads are present in the market. Given that spreads are common in trading environments and price-dependent orders are a core feature of the meta-transaction system, this represents a frequent occurrence during normal protocol usage.

### Recommendation
Consider modifying executePriceDependent() to decode the order side from req.selector and use the appropriate price for comparison. Buy orders should compare against ask prices, while sell orders should compare against bid prices.

```solidity
function executePriceDependent(PriceDependentRequest calldata req, bytes calldata signature)
    public
    payable
    returns (bytes memory)
{
    require(verifyPriceDependent(req, signature), KuruForwarderErrors.SignatureMismatch());
    require(msg.value >= req.value, KuruForwarderErrors.InsufficientValue());
    require(allowedInterface[req.selector], KuruForwarderErrors.InterfaceNotAllowed());
    executedPriceDependentRequest[keccak256(abi.encodePacked(req.from, req.nonce))] = true;

    (uint256 _currentBidPrice, uint256 _currentAskPrice) = IOrderBook(req.market).bestBidAsk();
    
    // Determine if this is a buy order based on selector
    bool isBuyOrder = req.selector == bytes4(keccak256("placeAndExecuteMarketBuy(uint96,uint256,bool,bool)"));
    uint256 relevantPrice = isBuyOrder ? _currentAskPrice : _currentBidPrice;
    
    require(
        (req.isBelowPrice && req.price < relevantPrice) || (!req.isBelowPrice && req.price > relevantPrice),
        PriceDependentRequestFailed(relevantPrice, req.price)
    );
    
    (bool success, bytes memory returndata) =
        req.market.call{value: req.value}(abi.encodePacked(req.selector, req.data, req.from));

    if (!success) {
        revert ExecutionFailed(returndata);
    }

    return returndata;
}
```
Notes
The bestBidAsk() function in the OrderBook correctly returns both bid and ask prices, but the forwarder implementation discards the ask price information. The test script in scripts/metatx/runPriceDependent.js demonstrates basic price-dependent execution but doesn't cover scenarios with wide spreads where this bug would manifest.


## [M-01] Vault share price front-running on deposit

### Summary
The KuruAMMVault contract's deposit() function is vulnerable to front-running attacks where MEV bots can manipulate the vaultBestAsk price immediately before user deposits to extract value. The deposit pricing relies on live, manipulable order book state without slippage protection or timing constraints. The severity is Medium due to repeatable value extraction from liquidity providers.

### Finding Description
The vault's deposit mechanism in _mintAndDeposit() calculates share prices using live order book data that can be manipulated within the same block. The function calls _returnNormalizedAmountAndPrice(true) which directly queries market.vaultBestAsk() for current pricing. This live price is then used in share conversion calculations through _convertToShares().

The vulnerability occurs because attackers can execute small trades to nudge the vaultBestAsk price in their favor, then immediately have user deposits execute at the manipulated price. The _convertToShares() function reads current reserves via totalAssets() and applies virtual rebalancing using the manipulated live pricing.

The deposit function lacks any slippage protection parameters - users cannot specify minimum shares out or maximum acceptable price bands. Additionally, there are no timing constraints or deadlines that would prevent delayed execution until favorable conditions appear.

### Impact Explanation
Front-runners can systematically extract small amounts of value from liquidity providers by manipulating vault pricing immediately before deposits. Each individual extraction may be small, but the attack can be repeated frequently across multiple transactions, creating a persistent drain on LP returns. The manipulation affects the fundamental share-to-asset conversion rate, directly impacting the value LPs receive for their deposits.

### Likelihood Explanation
This attack is highly feasible since it only requires the ability to execute small trades followed by triggering user deposits within the same block or transaction. MEV bots routinely perform such sandwich attacks in DeFi protocols. The economic incentive exists because even small percentage gains can be profitable when repeated frequently, especially given the lack of any protective mechanisms in the current implementation.

### Recommendation
Consider implementing slippage protection by adding minimum shares out parameters to the deposit function:
```solidity
function deposit(
    uint256 baseDeposit, 
    uint256 quoteDeposit, 
    address receiver,
    uint256 minSharesOut
) public payable nonReentrant returns (uint256) {
    // ... existing validation ...
    
    uint256 shares = _mintAndDeposit(baseDeposit, quoteDeposit, receiver);
    require(shares >= minSharesOut, KuruAMMVaultErrors.InsufficientSharesOut());
    
    return shares;
}
```
Alternatively, consider implementing time-weighted average pricing (TWAP) for share calculations, or block-scoped price snapshots to prevent intra-block manipulation. Adding deadline parameters would also help users control execution timing and prevent delayed execution during unfavorable market conditions.

### Notes
The virtual rebalancing mechanism in _returnVirtuallyRebalancedTotalAssets() compounds this vulnerability since attackers can influence both the base vaultBestAsk price and the virtual rebalancing calculations that adjust the effective reserves used in share conversions. The mixed rounding rules using both mulDivUp and regular mulDiv may also provide additional vectors for dust extraction through precise price manipulation.


## [L-01] Off-by-one on min size (strict ">")

### Summary
The OrderBook contract uses strict inequality (size > minSize) instead of inclusive inequality (size >= minSize) for size validation across all order placement functions. This rejects orders with size exactly equal to minSize, which appears unintentional based on error messages and initialization logic. The severity is Low due to minor impact on order placement boundaries.

### Finding Description
All order placement functions in the OrderBook contract contain an off-by-one error in size validation. The functions addBuyOrder(), addSellOrder(), addFlipBuyOrder(), addFlipSellOrder(), and addPairedLiquidity() use strict inequality checks like require(size > minSize, OrderBookErrors.SizeError()).

The error definition in OrderBookErrors.SizeError() states "Thrown when the size inputted is invalid, i.e, < minSize or > maxSize", suggesting inclusive behavior where minSize should be valid. However, the actual implementation rejects orders with size == minSize.

The contract initialization validates that minSize > 0 and maxSize > minSize, indicating minSize is intended as a meaningful lower bound.

### Impact Explanation
Users attempting to place orders with size exactly equal to minSize receive SizeError rejections, creating confusion since the minimum size should logically be inclusive. This affects order placement at the boundary but doesn't impact core trading functionality or cause fund loss.

### Likelihood Explanation
This issue occurs whenever users attempt to place orders with size exactly equal to minSize. While not every user will hit this exact boundary, it represents a consistent validation error that affects the expected behavior of minimum size limits.

### Recommendation
Consider changing the size validation to use inclusive inequality if minSize is intended to be a valid minimum order size:

- require(size > minSize, OrderBookErrors.SizeError());
+ require(size >= minSize, OrderBookErrors.SizeError());
This change should be applied consistently across all order placement functions. Alternatively, if exclusive behavior is intended, consider updating the error message and documentation to clarify that minSize represents an exclusive lower bound.


## [Info-01] Router passes wrong ETH value into markets

### Summary
The Router's anyToAnySwap() function incorrectly forwards raw _amount values as msg.value to OrderBook contracts without proper precision conversion. This causes multi-hop swaps involving native ETH to fail with NativeAssetInsufficient or NativeAssetSurplus errors because the OrderBook expects precisely converted native values. The severity is Medium due to broken functionality for native asset swaps.

### Finding Description
The Router's cross-market swap functionality contains a critical flaw in native ETH handling. In anyToAnySwap(), the function sets uint256 _value = _nativeSend[i] ? _amount : 0; and forwards this as {value: _value} to each market call.

The issue occurs because _amount represents token amounts in their respective precisions (base or quote), but OrderBook contracts expect native ETH values to be precisely converted according to market-specific precision parameters. The Router correctly converts amounts for calldata parameters using market precision factors like _amount * _marketParams.pricePrecision / 10 ** _marketParams.quoteAssetDecimals, but fails to apply the same conversion to the ETH value being forwarded.

Additionally, the Router updates _amount with each hop's output (_amount = _amountOut;) but continues using this updated amount as ETH value for subsequent native legs, which compounds the precision mismatch.

### Impact Explanation
Multi-hop swaps involving native ETH legs fail unpredictably when the forwarded msg.value doesn't match the OrderBook's expected converted amount. This breaks core routing functionality for native asset trades, significantly degrading user experience. Failed transactions may also strand ETH value in the Router contract if partial execution occurs before reversion.

### Likelihood Explanation
This issue affects any multi-hop swap route that includes native ETH legs, making it highly likely to occur during normal protocol usage. The precision mismatch is deterministic and will consistently cause failures when the raw amount doesn't align with the OrderBook's converted expectations.

### Recommendation
Consider implementing proper ETH value calculation for native legs by computing the exact msg.value each OrderBook requires. For native legs, calculate the converted amount using the same precision logic applied to calldata parameters, then sum the total required ETH and validate against the user's msg.value before execution.
```solidity
// Calculate exact ETH value needed for this hop
uint256 requiredEthValue = _nativeSend[i] ? 
    (_amount * _marketParams.pricePrecision / 10 ** _marketParams.quoteAssetDecimals) : 0;

// Use the calculated value instead of raw _amount
_amountOut = market.placeAndExecuteMarketBuy{value: requiredEthValue}(
    convertedAmount, 0, false, true
);
```
This approach ensures the forwarded ETH value matches the OrderBook's precision requirements and prevents reversion due to value mismatches.