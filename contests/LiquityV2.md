## LiquityV2
[Contest details](https://cantina.xyz/competitions/d86632df-ab33-4448-8198-64955eae6712)

### [Low-01] Users can prevent getting bad debt by withdrawing just before a liquidation

#### Summary
In Liquity V2, users can avoid redistributed bad debt from a liquidation by closing their trove just before the event, when the Stability Pool (SP) lacks sufficient BOLD. This increases the debt allocated to remaining troves in the same collateral market (e.g., WETH), unfairly shifting the burden to other borrowers.

#### Finding Description
When a trove’s Individual Collateral Ratio (ICR) falls below the Minimum Collateral Ratio (MCR, e.g., 110%) and the SP cannot cover its debt, Liquity V2 redistributes the liquidated trove’s debt (`redistBoldDebtGain`) and collateral (`redistCollGain`) across all active troves in the same collateral market (e.g., WETH). The debt is allocated proportionally based on each trove’s collateral share (e.g., a trove with 1% of WETH collateral receives 1% of the debt). A global tracker, `L_boldDebt`, records redistributed debt gas-efficiently, with troves applying pending debt on interaction (e.g., adjustTrove).

A user (e.g., Bob) can exploit this by:

- Detecting an impending liquidation (e.g., a WETH trove’s ICR < MCR, SP empty) using public data (trove debt, collateral, ETH/USD price, SP balance).
- Closing their trove via `closeTrove`, repaying debt and withdrawing collateral instantly, removing it from the active trove list.
- Waiting for the liquidation to redistribute debt across remaining troves.
- Optionally reopening a trove, paying a borrowing fee (e.g., 0.5%).
As long as the borrowing fee is lower than the bad debt that Bob would've received, this attack is profitable and will increase the bad debt going to the rest of the Troves. Also, Bob could not open the Trove again and avoid paying the borrowing fee, just avoiding the bad debt for free.

No malicious input is needed; instant `closeTrove` and public triggers enable the exploit.

#### Impact Explanation
The impact is Medium

#### Likelihood Explanation
The likelihood is medium as factors like sp not having enough to cover the debt

#### Proof of Concept
The PoC shows Bob avoiding redistributed debt by closing his WETH trove before a liquidation, leaving Alice with more debt:

Paste the code in the `redemption.t.sol`

```solidity
struct TestVars {
    uint256 liquidationAmount; // Charlie’s debt: 2,000 BOLD
    uint256 aliceColl; // Alice’s collateral: 10 WETH
    uint256 bobColl; // Bob’s collateral: 5 WETH
    uint256 charlieColl; // Charlie’s collateral: 1.222 WETH
    uint256 aliceDebt; // Alice’s debt: 10,000 BOLD
    uint256 bobDebt; // Bob’s debt: 5,000 BOLD
    uint256 price; // ETH price
    uint256 ADebt; // Alice’s initial debt
    uint256 BDebt; // Bob’s initial debt
    uint256 CDebt; // Charlie’s initial debt
    uint256 CInterest; // Charlie’s interest
    uint256 AFinalDebt; // Alice’s final debt
}
    function testTroveClosureBeforeLiquidationRedistribution__fromeo016() public {
    TestVars memory vars;
    ABCDEF memory troveIDs;

    // Initialize trove parameters
    vars.liquidationAmount = 2000e18; // 2,000 BOLD
    vars.aliceColl = 10e18; // 10 WETH
    vars.bobColl = 5e18; // 5 WETH
    vars.charlieColl = 1.222e18; // 1.222 WETH
    vars.aliceDebt = 10000e18; // 10,000 BOLD
    vars.bobDebt = 5000e18; // 5,000 BOLD

    // Set initial WETH price to $2,000
    priceFeed.setPrice(2000e18);

    // Alice opens trove (10 WETH, 10,000 BOLD)
    vm.startPrank(A);
    giveAndApproveCollateral(WETH, A, vars.aliceColl + 0.0375e18, address(borrowerOperations)); // +gas comp
    troveIDs.A = openTroveNoHints100pct(A, vars.aliceColl, vars.aliceDebt, MIN_ANNUAL_INTEREST_RATE);
    vm.stopPrank();

    // Bob opens trove (5 WETH, 5,000 BOLD)
    vm.startPrank(B);
    giveAndApproveCollateral(WETH, B, vars.bobColl + 0.0375e18, address(borrowerOperations));
    troveIDs.B = openTroveNoHints100pct(B, vars.bobColl, vars.bobDebt, MIN_ANNUAL_INTEREST_RATE);
    vm.stopPrank();

    // Charlie opens trove (1.222 WETH, 2,000 BOLD)
    vm.startPrank(C);
    giveAndApproveCollateral(WETH, C, vars.charlieColl + 0.0375e18, address(borrowerOperations));
    troveIDs.C = openTroveNoHints100pct(C, vars.charlieColl, vars.liquidationAmount, MIN_ANNUAL_INTEREST_RATE);
    vm.stopPrank();

    // Ensure SP is empty
    assertEq(stabilityPool.getTotalBoldDeposits(), 0, "SP should be empty");

    // Price drops to $1,800, making Charlies trove liquidatable (ICR ≈ 109.98% < 110%)
    priceFeed.setPrice(1800e18);
    (vars.price,) = priceFeed.fetchPrice();

    // Verify Charlies ICR < MCR
    assertLt(troveManager.getCurrentICR(troveIDs.C, vars.price), MCR, "Charlies ICR should be below MCR");
    assertGt(troveManager.getTCR(vars.price), CCR, "TCR should be above CCR");

    // Record initial debts
    vars.ADebt = troveManager.getTroveEntireDebt(troveIDs.A);
    vars.BDebt = troveManager.getTroveEntireDebt(troveIDs.B);
    vars.CDebt = troveManager.getTroveEntireDebt(troveIDs.C);

    // Bob closes his trove before liquidation
    vm.startPrank(B);
    deal(address(boldToken), B, vars.BDebt); // Provide Bob with BOLD to repay
    closeTrove(B, troveIDs.B);
    vm.stopPrank();

    // Verify Bobs trove is closed (status 3 = closed by owner)
    assertEq(uint8(troveManager.getTroveStatus(troveIDs.B)), uint8(ITroveManager.Status.closedByOwner), "Bobs trove should be closed");
    assertEq(troveManager.getTroveIdsCount(), 2, "Two troves should remain");

    // Liquidate Charlies trove
    vars.CInterest = troveManager.getTroveEntireDebt(troveIDs.C) - vars.liquidationAmount;
    liquidate(A, troveIDs.C);

    // Alice adjusts trove to apply redistributed gains (withdraw 1 wei BOLD)
    uint256 upfrontFeeA = predictOpenTroveUpfrontFee(50e18, MIN_ANNUAL_INTEREST_RATE);
    vm.startPrank(A);
    borrowerOperations.withdrawBold(troveIDs.A, 50e18, upfrontFeeA);
    vm.stopPrank();

    // Verify trove count reduced by 1
    assertEq(troveManager.getTroveIdsCount(), 1, "One trove should remain");

    // Verify SP remains empty
    assertEq(stabilityPool.getTotalBoldDeposits(), 0, "SP should remain empty");
    assertEq(stabilityPool.getCollBalance(), 0, "SP should have no collateral");

    // Check Alices debt increase (should receive all redistributed debt)

    vars.AFinalDebt = troveManager.getTroveEntireDebt(troveIDs.A);
    assertApproxEqAbs(
        (vars.AFinalDebt - vars.ADebt - 50e18 - upfrontFeeA) / 1e17,
        (vars.CDebt + vars.CInterest)/1e17,
        3,
        "Alices debt should increase by Charlies debt"
    );

    // Verify Bobs debt avoided
    assertEq(troveManager.getTroveEntireDebt(troveIDs.B), 0, "Bobs debt should be zero");

    // Verify no surplus collateral
    assertEq(collToken.balanceOf(address(collSurplusPool)), 0, "CollSurplusPool should be empty");
}
```

Then run
```bash
forge test --mt testTroveClosureBeforeLiquidationRedistribution__fromeo016
```

#### Recommendation
To mitigate this issue it's recommended to implement a timelock that ensures troves are not closed instantly but need to go through a withdrawal period (e.g. 1 day)

### [Low-02] Precision Loss in _calcInterest Function

The `_calcInterest` function in `TroveManager.sol` uses two sequential divisions `(/ ONE_YEAR / DECIMAL_PRECISION)`, introducing minor precision loss for small time periods (e.g., 13 seconds). While this could theoretically underreport interest, no practical attack path exists in the current system, as frequent trove adjustments are not incentivized or common. The issue is a minor implementation flaw with negligible impact.

Finding Description
The `_calcInterest` function is:
```solidity
function _calcInterest(uint256 _weightedDebt, uint256 _period) internal pure returns (uint256) {
    return _weightedDebt * _period / ONE_YEAR / DECIMAL_PRECISION;
}
```
#### Impact Explanation
- Economic Impact: Negligible loss (e.g., <1 BOLD/year on 3000 BOLD at 10% rate) due to infrequent small-period calculations.
- Scalability: Even large troves see minimal discrepancies
- Systemic Risk: No material effect on debt tracking or system stability.

#### Likelihood Explanation
- Likelihood: Low.
- Reason:
- Feasibility: Frequent adjustments (e.g., every 13s) cost ~6.6-13.3 ETH/day in gas, far outweighing potential savings (<0.01 BOLD/day).
- Incentive: No economic motive exists to exploit this, as gas costs dwarf the precision loss benefit.
- Occurrence: Normal usage involves longer intervals, avoiding the issue.

**Recommendation**
Update `_calcInterest` for improved precision:
```solidity
function _calcInterest(uint256 _weightedDebt, uint256 _period) internal pure returns (uint256) {
    return _weightedDebt * _period / (ONE_YEAR * DECIMAL_PRECISION);
}
```
Why:

Combines divisions into one step, preserving precision
Prevents theoretical underreporting, ensuring robustness even if system behavior changes (e.g., more frequent updates).
Impact: Negligible in current system, but enhances future-proofing.

### [Info-01] Liquidation rewards can be manipulated through stake inflation

#### Summary
The Liquity V2 TroveManager contract prevents liquidation of the last active trove due to a restrictive check in the `_closeTrove` function, allowing undercollateralized troves to persist and potentially causing protocol losses.

Finding Description
The `_closeTrove` function in the Liquity V2 TroveManager contract includes a call to `_requireMoreThanOneTroveInSystem`, which reverts with an `OnlyOneTroveLeft` error if the system has only one active trove (`TroveIds.length == 1`). This check is invoked during liquidation via the `_liquidate` function, which calls `_closeTrove` to close the liquidated trove. As a result, when a TroveManager instance has only one active trove, it cannot be liquidated, even if its Individual Collateralization Ratio (ICR) falls below the Minimum Collateralization Ratio (MCR, typically 110%).

This breaks the security guarantee of timely liquidation, which is critical to maintaining the protocol's solvency. Liquidation ensures that undercollateralized troves are closed, with their debt offset by the Stability Pool or redistributed to other troves, preventing losses from accumulating. By blocking liquidation of the last trove, an undercollateralized trove can remain active indefinitely, allowing the borrower to retain collateral and debt without consequence.

The issue is triggered as follows:

A single trove is opened in a TroveManager instance.
The collateral price drops, causing the trove's ICR to fall below the MCR.
A liquidator attempts to call liquidate on the TroveManager.
The `_liquidate` function processes the trove and calls `_closeTrove`.
`_closeTrove` invokes `_requireMoreThanOneTroveInSystem`, which checks `TroveIds.length`.
Since `TroveIds.length == 1`, the function reverts with `OnlyOneTroveLeft`, halting liquidation.
This bug does not require malicious input; it occurs naturally when a TroveManager has only one trove, which is a valid system state (e.g., in a new or low-usage TroveManager instance).

#### Impact Explanation
The inability to liquidate the last trove undermines the protocol's ability to manage risk and maintain solvency. An undercollateralized trove that cannot be liquidated may lead to:

- Loss of funds
- Delayed or permanent non-liquidation

The impact is not Critical because it is limited to `TroveManager` instances with a single trove, and the protocol may still function normally in instances with multiple troves. However, the potential for unrecoverable losses and the lack of a workaround make it a significant vulnerability.

#### Likelihood Explanation
The issue requires a `TroveManager` instance to have exactly one active trove, which may not be common in high-usage instances but is plausible.

The issue is triggered by a price drop making the trove undercollateralized, which is a realistic event in volatile markets. While not guaranteed to occur frequently, the combination of a single-trove state and price volatility makes the scenario sufficiently likely to warrant concern, especially in a decentralized protocol where usage patterns vary.

#### Proof of Concept
The following test integrates into the provided LiquidationsTest.sol to demonstrate the bug. It opens a single trove, simulates a price drop to make it undercollateralized, and attempts to liquidate it, expecting a revert with `OnlyOneTroveLeft`.

```solidity
function testImpossibleToLiquidateSingleTrove_fromeo016() public {
        // Step 1: Open a single trove with 2 WETH collateral
        uint256 collAmount = 2e18; // 2 WETH
        uint256 liquidationAmount = 2000e18; // 2000 BOLD
        uint256 initialPrice = 2000e18; // $2000 USD per WETH
        priceFeed.setPrice(initialPrice);

        vm.startPrank(A);
        uint256 ATroveId = borrowerOperations.openTrove(
            A,
            0,
            collAmount,
            liquidationAmount,
            0,
            0,
            MIN_ANNUAL_INTEREST_RATE,
            1000e18,
            address(0),
            address(0),
            address(0)
        );
        console.log("Trove opened with ID: %d, Collateral: %d WETH, Debt: %d BOLD", 
            ATroveId, collAmount / 1e18, liquidationAmount / 1e18);
        vm.stopPrank();

        // Verify only one trove exists
        uint256 trovesCount = troveManager.getTroveIdsCount();
        console.log("Trove count: %d", trovesCount);
        assertEq(trovesCount, 1, "Expected exactly one trove");

        // Step 2: Simulate price drop to $1100 USD to make trove undercollateralized
        uint256 newPrice = 1100e18 - 1; // Just below threshold for liquidation
        priceFeed.setPrice(newPrice);
        vm.warp(block.timestamp + 1); // Ensure price is not cached
        (uint256 price,) = priceFeed.fetchPrice();
        console.log("New WETH price: %d USD (18 decimals)", price / 1e18);
        assertEq(price, newPrice, "Price not updated correctly");

        // Verify trove is undercollateralized (ICR < MCR)
        uint256 icr = troveManager.getCurrentICR(ATroveId, price);
        uint256 mcr = troveManager.get_MCR(); // Typically 110%
        console.log("Trove ICR: %d%, MCR: %d%", icr / 1e16, mcr / 1e16);
        assertLt(icr, mcr, "Trove should be undercollateralized");

        // Step 3: Attempt to liquidate the trove, expect revert
        vm.expectRevert(TroveManager.OnlyOneTroveLeft.selector);
        troveManager.liquidate(ATroveId);
        console.log("Liquidation attempt failed as expected due to OnlyOneTroveLeft");

        // Step 5: Verify trove is still active
        uint256 troveStatus = uint256(troveManager.getTroveStatus(ATroveId));
        console.log("Trove status after liquidation attempts: %d (1 = active)", troveStatus);
        assertEq(troveStatus, 1, "Trove should remain active");
    }
```
Run with:
```bash
forge test --match-test testImpossibleToLiquidateSingleTrove_fromeo016 -vv
```

Expected Logs
```
[PASS] testImpossibleToLiquidateSingleTrove_fromeo016() (gas: 788343)
Logs:
  Trove opened with ID: 65628221146741533321501266128210572575400591302682096234549983883574975902220, Collateral: 2 WETH, Debt: 2000 BOLD
  Trove count: 1
  New WETH price: 1099 USD (18 decimals)
  Trove ICR: 109%, MCR: 110%
  Liquidation attempt failed as expected due to OnlyOneTroveLeft
  Trove status after liquidation attempts: 1 (1 = active)
```
The test confirms that both liquidate revert when attempting to liquidate the only trove, leaving the undercollateralized trove active.

#### Recommendation
To fix the issue, the `_requireMoreThanOneTroveInSystem` check should be removed or modified to allow liquidation of the last trove.

OR

The protocol could adopt this strategy: maintain a protocol-owned trove with minimal debt and high ICR in each TroveManager to ensure at least two troves exist, allowing liquidation of user troves. This approach avoids code changes but requires operational overhead.