## Liquid Ron
[Contest Details](https://code4rena.com/audits/2025-01-liquid-ron)

### [Medium-01] Functions with `onlyOperator` modifier would still revert even if called by an operator

**Finding description and impact**

The `onlyOperator` modifier would revert if the `operator[msg.sender] == true`, meaning an operator wouldn't be able to call the functions that use the modifier which breaks the functionality of the contract

Proof of Concept
Run this test
```solidity
function testRevertIfOperator(uint88 _amount) public {
        vm.assume(_amount >= 0.01 ether);
        liquidRon.deposit{value: _amount}();
        uint256 delegateAmount = _amount / 17;
        uint256[] memory amounts = new uint256[](5);
        for (uint256 i = 0; i < 5; i++) {
            amounts[i] = delegateAmount;
        }
        uint256 totalAsset = liquidRon.totalAssets();
        address USER = makeAddr("USER");
        liquidRon.updateOperator(USER, true);
        vm.startPrank(USER);
        vm.expectRevert(LiquidRon.ErrInvalidOperator.selector);
        liquidRon.delegateAmount(0, amounts, consensusAddrs);
        vm.expectRevert(LiquidRon.ErrInvalidOperator.selector);
        liquidRon.delegateAmount(1, amounts, consensusAddrs);
        vm.expectRevert(LiquidRon.ErrInvalidOperator.selector);
        liquidRon.delegateAmount(2, amounts, consensusAddrs);
        assertEq(liquidRon.totalAssets(), totalAsset);
        vm.stopPrank();
    }
```
Recommendation
```solidity
/// @dev Modifier to restrict access of a function to an operator or owner
    modifier onlyOperator() {
-        if (msg.sender != owner() || operator[msg.sender]) revert ErrInvalidOperator();
+        if (msg.sender != owner() || operator[msg.sender] == false) revert ErrInvalidOperator();
        _;
    }
```

### [Medium-02] Inaccurate Fee Accrual When `OperatorFee` is set to new values

**Finding description and impact**

The owner can update the `operatorFee` with the `setOperatorFee` function in the `LiquidRon.sol`. However, the current implementation does not account for accrued fees before the update, potentially leading to incorrect fee calculation. Operator fee might have been pending for the last set of `operatorFee` param and needs to be claimed before new params are set , else fee not accrued is lost forever.

**Recommended mitigation steps**

Call the `fetchOperatorFee`() before updating the management fee params.


### [Low-01] No upperbound for validators (Out of gas risk)

**Finding description and impact**

The `pruneValidatorList` iterates through the total number of validators in the validator list. This could lead to an out of gas error if the number of validators is too high. This is particularly concerning as the number of proxies also increase over time, as the loop would iterate through a large validator list and iterate through the proxy list for each member of the validator list. For each member in the validator list, it goes through the proxy list twice.

Proof of Concept
The number of validators increase, so does the number of proxies
An attempt to prune the validator list is made
The function iterates through all members of the validator list and further iterates through the proxy list for each for each member of the validator list
This leads to excessive gas usage causing out of gas errors

