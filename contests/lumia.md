# LUMIA
[Contest Details](https://dashboard.hackenproof.com/user/programs/lumia-smart-contract-audit-contest)


## [C-01] Direct stake withdrawals fail to return funds

### Summary
Direct withdrawal requests in the direct staking strategy never release funds back to the requester because no component performs the transfer after an exit is claimed. The severity is Critical, since the behavior causes a permanent freezing of user funds while the protocol retains custody of the tokens. This outcome undermines direct-stake exits and prevents users from recovering their stake.

### Finding Description
Direct stake positions remain custodied by the diamond contract, so redemptions rely on DirectStakeStrategy.claimExit() to hand assets to the receiver. The function only marks exitRequests[receiver_] as claimed, returns the recorded amountClaimed, and emits ExitClaimed without calling CurrencyHandler to transfer the underlying ERC-20 or native asset. DepositFacet.claimWithdraws() deletes the pending request before delegating to the strategy and assumes that the strategy will transfer the payout, so it merely updates accounting and emits WithdrawClaimed. Because neither function moves funds, the diamond contract continues to hold the user's stake after a withdrawal claim completes, leaving the caller with nothing despite a successful claim flow.

### Impact Explanation
Users who initiate direct-stake withdrawals can never receive their tokens, resulting in a permanent freezing of deposited assets that keeps direct vault liquidity locked inside the protocol.

### Likelihood Explanation
Any direct-stake participant can reproduce the issue by performing a normal withdrawal through the publicly exposed flow, and the attack requires no special privileges or complex setup.

### Recommendation
Consider updating DirectStakeStrategy.claimExit() to invoke the appropriate CurrencyHandler transfer method so that the recorded stake amount is remitted to receiver_. One approach could be to have DepositFacet.claimWithdraws() detect direct strategies and explicitly transfer the returned amount before emitting WithdrawClaimed to ensure custody exits always result in funds leaving the diamond.

### Validation steps
`test/unit/DirectStake.test.ts` contains the case titled "claiming direct stake withdrawals never returns funds to the receiver", which walks through a full withdrawal flow and asserts that:

The diamond accumulates the user's stake at deposit time.
After the request unlocks, DepositFacet.claimWithdraws() succeeds yet the receiver's USDC balance is unchanged.
The diamond still holds the stake amount even though the claim completed.



## [M-01] Protocol fee withdrawals permanently inflate pendingExitStake

### Summary
Protocol fee harvesting reuses the user exit queue without unwinding the pending counter. Each successful fee withdrawal permanently increases pendingExitStake, reducing the available exit capacity for real users. Severity: High, because routine fee operations can freeze legitimate withdrawals.

### Finding Description
AllocationFacet.report() invokes _leave() to withdraw protocol fees, which increments stakeInfo[strategy].pendingExitStake before queuing the claim through IDeposit.queueWithdraw(). When the queued fee withdrawal is later processed in DepositFacet.claimWithdraws(), the feeWithdraw branch emits FeeWithdrawClaimed but skips the bookkeeping steps that subtract the stake value from pendingExitStake. As a result, every fee harvest leaves behind a phantom pending amount even though the claim settled successfully. Once enough harvests occur, pendingExitStake approaches or exceeds totalStake, so _leave() computes availableStake = totalStake - pendingExitStake as zero and subsequent user withdrawals revert or produce negligible allocations despite adequate strategy liquidity.

### Impact Explanation
Legitimate users may be unable to exit because the system believes all stake is already pending withdrawal, resulting in a denial of service for withdrawals.

### Likelihood Explanation
Fee harvesting is a routine management action, so the invariant drifts toward failure under normal protocol operation without requiring malicious behavior.

### Recommendation
Consider isolating protocol fee bookkeeping from user exits so that pendingExitStake only tracks user requests; for example, avoid incrementing it for fee withdrawals or add a balancing decrement when feeWithdraw claims finalize while tracking protocol fees in a dedicated counter.

### Validation steps
Paste in `test/unit/Staking.test.ts`

```typescript
it("fee claims should unwind pending exit stake", async function () {
        const { hyperStaking, reserveStrategy1, signers } = await loadFixture(deployHyperStaking);
        const { deposit, allocation } = hyperStaking;
        const { alice, bob: feeRecipient, vaultManager, strategyManager } = signers;

        await allocation.connect(vaultManager).setFeeRecipient(reserveStrategy1, feeRecipient);
        const feeRate = parseEther("0.1");
        await allocation.connect(vaultManager).setFeeRate(reserveStrategy1, feeRate);

        const stakeAmount = parseEther("10");
        await deposit.stakeDeposit(reserveStrategy1, alice, stakeAmount, { value: stakeAmount });

        const beforeReport = await allocation.stakeInfo(reserveStrategy1);
        expect(beforeReport.pendingExitStake).to.equal(0);

        const assetPrice = await reserveStrategy1.assetPrice();
        await reserveStrategy1.connect(strategyManager).setAssetPrice(assetPrice * 12n / 10n);

        const expectedRevenue = await allocation.checkRevenue(reserveStrategy1);
        expect(expectedRevenue).to.be.gt(0n);
        const expectedFeeAmount = expectedRevenue * feeRate / parseEther("1");
        expect(expectedFeeAmount).to.be.gt(0n);

        await allocation.connect(vaultManager).report(reserveStrategy1);

        const afterReport = await allocation.stakeInfo(reserveStrategy1);
        expect(afterReport.pendingExitStake).to.equal(expectedFeeAmount);

        const claimId = await shared.getLastClaimId(deposit, reserveStrategy1, feeRecipient);
        await deposit.connect(feeRecipient).claimWithdraws([claimId], feeRecipient);

        const afterClaim = await allocation.stakeInfo(reserveStrategy1);
        expect(afterClaim.pendingExitStake).to.equal(0);
      });
```

Run

```bash
npx hardhat test test/unit/Staking.test.ts --no-compile --grep "fee claims should unwind pending exit stake"
```
The test asserts that a protocol-fee claim must return pendingExitStake to zero, but on current contracts the value remains equal to the fee amount after claimWithdraws, so the final expectation fails. That failed assertion is the direct symptom of the bug described in the report, showing that fee harvests leave phantom pending stake behind.


