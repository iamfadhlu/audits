### [H-01] Incorrect amount used in `RToken::_mint` and `RToken::burn` functions

#### Summary
In `RToken::mint` and `RToken::burn`, the amount being minted and burned is incorrect because it is denominated in units of the underlying asset instead of RTokens. This discrepancy leads to errors in the minting and burning process.

Additionally, the `balanceIncrease` isn't added to the amount minted so the interest accrued are not gotten by the user.

#### Proof of Concept
```solidity
     * @notice Mints RToken to a user
     * @param caller The address initiating the mint
     * @param onBehalfOf The recipient of the minted tokens
     * @param amountToMint The amount of tokens to mint (in underlying asset units)
     * @param index The liquidity index at the time of minting
     * @return A tuple containing:
     *         - bool: True if this is the first mint for the recipient, false otherwise
     *         - uint256: The amount of scaled tokens minted
     *         - uint256: The new total supply after minting
     *         - uint256: The amount of underlying tokens minted
     */
    function mint(
        address caller,
        address onBehalfOf,
        uint256 amountToMint,
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256, uint256) {
        if (amountToMint == 0) {
            return (false, 0, 0, 0);
        }
        uint256 amountScaled = amountToMint.rayDiv(index);
        if (amountScaled == 0) revert InvalidAmount();

        uint256 scaledBalance = balanceOf(onBehalfOf);
        bool isFirstMint = scaledBalance == 0;

        uint256 balanceIncrease = 0;
        if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
            balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
        }

        _userState[onBehalfOf].index = index.toUint128();

@>      _mint(onBehalfOf, amountToMint.toUint128());

        emit Mint(caller, onBehalfOf, amountToMint, index);

        return (isFirstMint, amountToMint, totalSupply(), amountScaled); // @audit 
    }
```
The amount to be minted should be in `RToken` units, not the underlying asset units. This discrepancy leads to errors in the minting process.

Furthermore, the amount being burned in `_burn` is incorrect because the amount passed in `_burn` is denominated in units of the underlying asset.
```solidity
/**
     * @notice Burns RToken from a user and transfers underlying asset
     * @param from The address from which tokens are burned
     * @param receiverOfUnderlying The address receiving the underlying asset
     * @param amount The amount to burn (in underlying asset units)
     * @param index The liquidity index at the time of burning
     * @return A tuple containing:
     *         - uint256: The amount of scaled tokens burned
     *         - uint256: The new total supply after burning
     *         - uint256: The amount of underlying asset transferred
     */
    function burn(
        address from,
        address receiverOfUnderlying,
        uint256 amount,
        uint256 index
    ) external override onlyReservePool returns (uint256, uint256, uint256) {
        if (amount == 0) {
            return (0, totalSupply(), 0);
        }

        uint256 userBalance = balanceOf(from);  

        _userState[from].index = index.toUint128();

        if(amount > userBalance){
            amount = userBalance;
        }

        uint256 amountScaled = amount.rayMul(index);

        _userState[from].index = index.toUint128();

@>      _burn(from, amount.toUint128());

        if (receiverOfUnderlying != address(this)) {
            IERC20(_assetAddress).safeTransfer(receiverOfUnderlying, amount);
        }

        emit Burn(from, receiverOfUnderlying, amount, index);

@>      return (amount, totalSupply(), amount); // @audit wrong return value  
    }
```
#### Impact
This leads to incorrect mint and burn amounts

It returns the wrong value which could mislead integrators about the actual amount of RTokens minted / burned, potentially causing accounting errors in protocols integrating with this contract

#### Tools Used
Manual Review

#### Recommendations
`amountScaled` should be used in the _mint function instead of `amountToMint` to ensure that the correct amount of RTokens is minted and the correct values are returned in the function.

```solidity
/**
     * @notice Mints RToken to a user
     * @param caller The address initiating the mint
     * @param onBehalfOf The recipient of the minted tokens
     * @param amountToMint The amount of tokens to mint (in underlying asset units)
     * @param index The liquidity index at the time of minting
     * @return A tuple containing:
     *         - bool: True if this is the first mint for the recipient, false otherwise
     *         - uint256: The amount of scaled tokens minted
     *         - uint256: The new total supply after minting
     *         - uint256: The amount of underlying tokens minted
     */
    function mint(
        address caller,
        address onBehalfOf,
        uint256 amountToMint,
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256, uint256) {
        if (amountToMint == 0) {
            return (false, 0, 0, 0);
        }
        uint256 amountScaled = amountToMint.rayDiv(index);
        if (amountScaled == 0) revert InvalidAmount();

        uint256 scaledBalance = balanceOf(onBehalfOf);
        bool isFirstMint = scaledBalance == 0;

        uint256 balanceIncrease = 0;
        if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
            balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
        }

        _userState[onBehalfOf].index = index.toUint128();
+       amountToMint += balanceIncrease;
+       amountScaled = amountToMint.rayDiv(index);

-       _mint(onBehalfOf, amountToMint.toUint128());
+       _mint(onBehalfOf, amountScaled.toUint128());

        emit Mint(caller, onBehalfOf, amountToMint, index);

-       return (isFirstMint, amountToMint, totalSupply(), amountScaled);
+       return (isFirstMint, amountScaled, totalSupply(), amountToMint);
    }
```
`amountScaled` should be used in the _burn function instead of amount to ensure that the correct amount of RTokens is burned and the correct values are returned in the function.

### [H-02] Incorrect Debt Scaling Prevents Borrower Liquidation

#### Summary

The `liquidateBorrower` function incorrectly scales the user's debt a second time, causing an inaccurate balance check. This results in unnecessary reverts, preventing liquidations from occurring as intended.

#### Vulnerability Details

The issue occurs due to **double scaling** of `userDebt`:

```solidity
uint256 userDebt = lendingPool.getUserDebt(userAddress); // Already in underlying asset units

uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt()); // @audit further scaling when it's already in correct units
```

* The function `getUserDebt` already returns the **debt in underlying asset units**, meaning **no further scaling** is required.

* The additional scaling (`rayMul(userDebt, lendingPool.getNormalizedDebt())`) inflates the value, leading to an **incorrect balance check**:

```solidity
if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance(); // Always reverts due to over-scaling
```

* Since `scaledUserDebt` is **artificially higher than it should be**, the function **always reverts**, blocking valid liquidations.

#### Impact

This issue **completely prevents borrower liquidations**, leading to:

* Accumulation of bad debt, as liquidations are blocked.

* Risk to protocol solvency, since positions that should be liquidated remain open.

* Stability Pool funds not being utilized\*\*, affecting ecosystem health.

#### Tools Used

Manual Review

#### Recommendations

**Fix the incorrect scaling**\
Remove the unnecessary `rayMul` operation:

```solidity
uint256 userDebt = lendingPool.getUserDebt(userAddress); // Correct units, no further scaling needed
```

And update the balance check:

```solidity
if (crvUSDBalance < userDebt) revert InsufficientBalance(); // Now correctly compares against available funds
```

And update the approval amount:

```solidity
bool approveSuccess = crvUSDToken.approve(address(lendingPool), userDebt); // Approve the correct amount
```

### [H-03] Incorrect amount transferred from user during debt repayment

#### Summary

The `_repay` and `finalizeLiquidation` functions in the `LendingPool` smart contract incorrectly transfer scaled amount of tokens, `amountScaled`, instead of the amount of underlying, `amountBurned`, returned from the `DebtToken::burn`, which is meant to be the amount of DebtTokens burned in units of the underlying asset. This results in a mismatch between the debt reduction and the actual amount repaid, leading to financial inconsistencies in the protocol.

#### Vulnerability Details

In the `_repay` and `finalizeLiquidation` function, the following line executes the transfer of reserve assets:

```solidity
// Transfer reserve assets from the caller (msg.sender) to the reserve
IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
```

Issue:

* The function transfers `amountScaled`, which is derived from the internal debt token calculations.

* The expected behavior is to transfer the actual repayment amount, `amountBurned`, returned from the `DebtToken::burn`, which is meant to be the reserveAsstet worth of the debtTokens burned, taking account of the accrued interest

* `amountScaled` does not match the expected underlying token amount due to scaling differences, the repayment would be inaccurate, leading to inconsistencies between debt accounting and actual asset movements.

#### Impact

* Accounting Inconsistencies: The protocol's debt and collateral accounting may become unreliable, affecting liquidation calculations and reserve balances.

* Potential Exploitation: If the issue allows users to manipulate repayment calculations, it could be used for unintended financial gains.

#### Tools Used

Manual Review

#### Recommendations

Ensure that the amount transferred matches the repayment amount with the interest accrued accounted for in units of the underlying assets. Consider the following changes:

```diff
    // Transfer reserve assets from the caller (msg.sender) to the reserve
-   IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
+   IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountBurned);
```

### [H-04] Wrong amount used in `DebtToken::mint`

#### Summary

In `DebtToken::mint`, the amount minted is wrong. The `amountToMint` is in units of underlying assets instead of scaled units(debtToken). This leads to the wong amount of debtToken being minted, making a user to mint incorrect amount of debtToken than they should have.

#### Vulnerability Details

The issue arises because:

`amount` is in underlying asset units
`balanceIncrease` is calculated in underlying asset units
These unscaled values are directly used in `_mint`
However, `_mint` should receive scaled units (DebtToken units)

#### Impact

* Users receive incorrect amounts of debt tokens when they mint debt tokens.

* System accounting becomes inaccurate

* Protocol's debt tracking becomes unreliable

* Could lead to under-collateralization or excessive borrowing

* Protocol's economic model becomes unstable

#### Tools Used

Manual Review

#### Recommendations

```diff
/**
     * @notice Mints debt tokens to a user
     * @param user The address initiating the mint
     * @param onBehalfOf The recipient of the debt tokens
     * @param amount The amount to mint (in underlying asset units)
     * @param index The usage index at the time of minting
     * @return A tuple containing:
     *         - bool: True if the previous balance was zero
     *         - uint256: The amount of scaled tokens minted
     *         - uint256: The new total supply after minting
     */
    function mint(
        address user,
        address onBehalfOf,
        uint256 amount,
        uint256 index
    )  external override onlyReservePool returns (bool, uint256, uint256) {
        if (user == address(0) || onBehalfOf == address(0)) revert InvalidAddress();
        if (amount == 0) {
            return (false, 0, totalSupply());
        }

        uint256 amountScaled = amount.rayDiv(index);
        if (amountScaled == 0) revert InvalidAmount();

        uint256 scaledBalance = balanceOf(onBehalfOf);
        bool isFirstMint = scaledBalance == 0;

        uint256 balanceIncrease = 0;
        if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
            balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
        }

        _userState[onBehalfOf].index = index.toUint128();

        uint256 amountToMint = amount + balanceIncrease;
+       amountToMint = amountToMint.rayDiv(index);

        _mint(onBehalfOf, amountToMint.toUint128());

        emit Transfer(address(0), onBehalfOf, amountToMint);
        emit Mint(user, onBehalfOf, amountToMint, balanceIncrease, index);

-        return (scaledBalance == 0, amountToMint, totalSupply());
+        return (isFirstMint, amountToMint, totalSupply());
    }
```


### [M-01] Incorrect Boost Multiplier Calculation in getBoostMultiplier

#### Summary

The function `getBoostMultiplier` incorrectly calculates the boost multiplier due to a flawed formula. Instead of properly determining the multiplier based on the user’s veToken balance, it incorrectly derives a value that does not correctly reflect the intended boost mechanics.

#### Vulnerability Details

In `getBoostMultiplier`, the boost multiplier is computed as:

```solidity
uint256 baseAmount = userBoost.amount * 10000 / MAX_BOOST;
return userBoost.amount * 10000 / baseAmount;
```

* The `baseAmount` calculation incorrectly scales `userBoost.amount`, but the final division essentially cancels out the intended effect, returning an incorrect boost value.

* If `userBoost.amount == MAX_BOOST`, then `baseAmount = 10000`, making the function return `userBoost.amount`, which is not a valid multiplier in basis points.

By contrast, the correct boost calculation is seen in `calculateBoost`:

```solidity
uint256 votingPowerRatio = (veBalance * 1e18) / totalVeSupply;
uint256 boostRange = params.maxBoost - params.minBoost;
uint256 boost = params.minBoost + ((votingPowerRatio * boostRange) / 1e18);
```

This correctly computes the boost based on the user's veToken balance relative to the total supply.

#### Impact

* The incorrect calculation could lead to **misrepresented boost values**, potentially allowing **incorrect reward distributions** or **unintended behavior** in the system.

#### Tools Used

Manual Review

#### Recommendations

* Update `getBoostMultiplier` to use a calculation similar to `calculateBoost`, ensuring that the boost is based on the user’s veToken balance relative to total veToken supply.

### [M-02] Incorrect Reduction of Pool Boost on Delegation Removal

#### Summary

The `removeBoostDelegation` function incorrectly decreases `poolBoost.totalBoost` and `poolBoost.workingSupply` when a delegation is removed, even though `delegateBoost` does not modify these values initially. This leads to an **artificial reduction** in pool boost metrics, which impacts reward calculations.

#### Vulnerability Details

* In `delegateBoost`, when a user delegates their boost, `poolBoost.totalBoost` and `poolBoost.workingSupply` remain **unchanged** (correct behavior).

* However, in `removeBoostDelegation`, the delegated amount is **subtracted** from these pool-wide metrics:

  ```solidity
  if (poolBoost.totalBoost >= delegation.amount) {
      poolBoost.totalBoost -= delegation.amount; // @issue: Should not modify totalBoost
  }
  if (poolBoost.workingSupply >= delegation.amount) {
      poolBoost.workingSupply -= delegation.amount;
  }
  ```

* Since delegation **does not increase these values** initially, subtracting the amount upon removal leads to an incorrect **deflation** of pool-wide boost values.

**Code References**

**Correct Behavior (Delegation Does Not Modify Pool Boost)**

```solidity
delegation.amount = amount;
delegation.expiry = block.timestamp + duration;
delegation.delegatedTo = to;
delegation.lastUpdateTime = block.timestamp;
```

**Incorrect Behavior (Delegation Removal Modifies Pool Boost)**

```Solidity
if (poolBoost.totalBoost >= delegation.amount) {
    poolBoost.totalBoost -= delegation.amount; // @issue: Incorrect modification
}
if (poolBoost.workingSupply >= delegation.amount) {
    poolBoost.workingSupply -= delegation.amount; // @issue: Incorrect modification
}
poolBoost.lastUpdateTime = block.timestamp;
```

#### Impact

* **Artificial Reduction of Pool Boost:**

  * The total boost available to the pool is **incorrectly reduced** when delegation is removed.

  * This can lead to inaccurate reward calculations, as the pool appears to have less boost than it actually does.

* **Potential Exploit Scenarios:**

  * Attackers could repeatedly delegate and remove delegations to **artificially suppress** the pool's total boost, potentially impacting reward distributions.

#### Tools Used

Manual Review

#### Recommendations

Consider removing the incorrect Pool Boost Modifications

### [M-03] Incorrect Boost Application Due to Fixed Base Amount

#### Summary

The `_calculateBoost` function incorrectly applies the boost multiplier to a fixed amount (`10000`) instead of the user's actual stake. While the boost multiplier is correctly determined based on the user's veToken balance, the boosted amount is always derived from this fixed value, leading to inaccurate reward calculations.

#### Vulnerability Details

In the `updateUserBoost` function, the new boost is determined using:

```solidity
uint256 newBoost = _calculateBoost(user, pool, 10000); // Base amount
```

This calls `_calculateBoost`, which calculates the boost multiplier correctly but applies it to a hardcoded `10000` instead of the user's actual balance.

Within `calculateTimeWeightedBoost`:

```solidity
boostedAmount = (amount * boostBasisPoints) / 10000;
```

Since `amount` is always `10000`, the boosted amount is effectively just `boostBasisPoints`, rather than being proportional to the user's actual stake. This means the boost is not properly scaled to individual user balances.

#### Impact

* The boost multiplier is correctly calculated but does not apply to the user's real stake, leading to miscalculated rewards.

* Users with larger veToken balances do not receive appropriately scaled boosts, while smaller holders get disproportionate boosts.

* The intended mechanics of the boost system are compromised, leading to unfair reward distribution.

#### Tools Used

Manual Review

#### Recommendations

* Modify `_calculateBoost` to use the user's actual stake instead of `10000`.

* Ensure `calculateTimeWeightedBoost` applies the boost multiplier to the user's real balance.


### [M-04] Incorrect `workingSupply` Update in `updateUserBoost`

#### Summary

The function `updateUserBoost` incorrectly assigns the `workingSupply` of the pool to `newBoost`, which is the **boost amount of a single user**. This causes `workingSupply` to be overwritten incorrectly instead of maintaining a cumulative value across all users in the pool. This results in **misleading accounting data** for the pool's working supply.

#### Vulnerability Details

* The `workingSupply` variable is meant to represent the **total** effective supply of the pool, including all users' boosts.

* However, in `updateUserBoost`, it is directly set to `newBoost`, which only accounts for the boost of the **currently updated user**.

* This means the previous working supply data is **overwritten**, likely leading to incorrect calculations in future reward distributions.

* As more users update their boosts, `workingSupply` will reflect only the latest updated user's boost, **not the sum of all users' boosts**.

* Since `workingSupply` is not used for boost calculations, this does **not** impact reward distribution but leads to **incorrect accounting data**.

#### Impact

* **Misleading Pool Data:** The `workingSupply` field will **not** correctly represent the pool's true boosted supply, which could cause confusion in off-chain tracking and analytics.

* **Potential UI/Reporting Issues:** Any frontend, dashboard, or analytics tool relying on `workingSupply` may display **incorrect pool statistics**.

* **Operational Inefficiencies:** Incorrect tracking could lead to **misinterpretations** of liquidity and boost distribution, affecting governance or future protocol decisions.

#### Tools Used

Manual Review

#### Recommendations

Instead of directly setting `workingSupply = newBoost`, update it incrementally like `totalBoost` to reflect **cumulative** changes.

### [M-05] _depositIntoVault: Curve Deposits Always Revert Due to Incorrect Asset Ownership

#### Summary

A critical bug in the `_depositIntoVault` function prevents liquidity deposits into the Curve vault. The function incorrectly assumes that the `LendingPool` contract holds the assets, when in reality, the `RToken` contract does. This causes the deposit function to **always revert**, breaking the intended yield-generating mechanism.

#### Vulnerability Details

```Solidity
function depositIntoVault(uint256 amount) internal {
    IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
    curveVault.deposit(amount, address(this)); // ❌ Incorrect sender, LendingPool does not hold assets
    totalVaultDeposits += amount;
}
```

### Root Cause:

* The LendingPool contract calls `curveVault.deposit(amount, address(this))`.

* However, the **LendingPool does not hold the reserve assets**—the RToken contract does.

* As a result, `curveVault.deposit` **always fails**, preventing any deposits.

### Expected Behavior:

* The deposit should be made **from the RToken contract**, not the LendingPool contract.

* The RToken contract should approve and execute the deposit, ensuring assets are correctly moved into the Curve vault.

#### Impact

* **Deposits into the Curve vault are impossible**, completely breaking this functionality.

* **Users miss out on yield generation**, reducing the protocol’s efficiency.

* **Funds remain stuck in the RToken contract**, never reaching the Curve vault.

* **Yield strategies relying on Curve are disrupted**, leading to potential financial losses.

#### Tools Used

* Manual Code Review

#### Recommendations

* Fix: Use the RToken Contract for Deposits

Modify `_depositIntoVault` to ensure the `RToken` contract executes the deposit:

```solidity
function _depositIntoVault(uint256 amount) internal {
    IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
    curveVault.deposit(amount, reserve.reserveRTokenAddress); // ✅ Use the RToken contract for deposit
    totalVaultDeposits += amount;
}
```

### [M-06] Buffer withdrawn to wrong address in `_rebalanceLiquidity`

#### Summary

The `_rebalanceLiquidity` and `_ensureLiquidity` functions incorrectly withdraw tokens to the LendingPool address instead of the reserveRTokenAddress when rebalancing the buffer or ensuring liquidity, leading to potential accounting errors and redundant withdrawals.

#### Vulnerability Details

The issue occurs in the `_rebalanceLiquidity` function when `currentBuffer < desiredBuffer`:

```solidity
else if (currentBuffer < desiredBuffer) {
    uint256 shortage = desiredBuffer - currentBuffer;
    // Withdraw shortage from the Curve vault
@>  _withdrawFromVault(shortage);
}
```

And in `_ensureLiquidity` if `(availableLiquidity < amount)`:

```solidity
if (availableLiquidity < amount) {
            uint256 requiredAmount = amount - availableLiquidity;
            // Withdraw required amount from the Curve vault
@>      _withdrawFromVault(requiredAmount);
        }
```

The `_withdrawFromVault` function withdraws the `shortage` / `requiredAmount` from the Curve vault and transfers it to the LendingPool address:

```solidity
function _withdrawFromVault(uint256 amount) internal {
@>  curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0));
    totalVaultDeposits -= amount;
    }
```

Tokens are withdrawn to LendingPool address instead of reserveRTokenAddress
`currentBuffer` and `availableLiquidity` check balance at reserveRTokenAddress but withdrawals don't go there
This mismatch means subsequent calls will keep detecting a shortage
Results in repeated withdrawals since withdrawals never increases at reserveRTokenAddress

#### Impact

* Buffer mechanism becomes ineffective as tokens aren't stored in correct location

* Repeated unnecessary withdrawals from vault on each rebalance and ensureLiquidity call

* Increased gas costs from redundant operations

* Potential depletion of vault liquidity through repeated withdrawals

* Break in system invariants around buffer and liquidity maintenance

#### Tools Used

Manual review

#### Recommendations

* Modify \_withdrawFromVault to withdraw to reserveRTokenAddress:

```solidity
    function _withdrawFromVault(uint256 amount) internal {
    // Change withdrawal destination to reserveRTokenAddress
    curveVault.withdraw(amount, reserve.reserveRTokenAddress);
}
```

* Add validation to ensure tokens arrive at correct address:

```solidity
    function _withdrawFromVault(uint256 amount) internal {
    uint256 balanceBefore = IERC20(reserve.reserveAssetAddress).balanceOf(reserve.reserveRTokenAddress);
    curveVault.withdraw(amount, reserve.reserveRTokenAddress);
    uint256 balanceAfter = IERC20(reserve.reserveAssetAddress).balanceOf(reserve.reserveRTokenAddress);
    require(balanceAfter - balanceBefore == amount, "Withdrawal failed");
}
```







