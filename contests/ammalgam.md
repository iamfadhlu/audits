### [H-01] `onlyOwner` on `isPluginAllowed` Prevents Plugin Addition for Non-Owners

#### Summary

The `isPluginAllowed` function in `PluginRegistry.sol` is restricted by `onlyOwner`, which prevents non-owner users from successfully calling `ERC20Base.addPlugin`. Since `addPlugin` depends on `isPluginAllowed`, any non-owner attempting to register a plugin will revert, effectively breaking the plugin extensibility mechanism for all non-owners.

---

#### Finding Description

In `PluginRegistry.sol`, the check for plugin approval is defined as:

```solidity
function isPluginAllowed(address plugin) public view onlyOwner returns (bool) {
    return allowedPlugins[plugin];
}
```

Because of the `onlyOwner` modifier, only the registry owner can call this function.

However, in `ERC20Base.sol`, the public `addPlugin` method—intended for general users—depends on it:

```solidity
function addPlugin(address plugin) public override {
    if (pluginRegistry.isPluginAllowed(plugin)) {
        super._addPlugin(msg.sender, plugin);
    }
}
```

When a non-owner attempts to call `addPlugin`, the underlying `isPluginAllowed` call reverts due to the access restriction. As a result, plugin integration is completely blocked for regular users, undermining the design intent of allowing extensibility.

---

#### Impact Explanation

* **Medium functional impact**: No direct fund loss, but a core feature of the system—extensible plugins—becomes unusable for non-owner users.
* **Protocol degradation**: Prevents integrations or extensions that rely on public plugin registration.
* **Broken guarantees**: Contradicts the expectation that `addPlugin` can be used by any token holder.

Overall severity is classified as **High**, since it blocks a critical system feature across all deployments.

---

#### Likelihood Explanation

* **High likelihood**:

  * Every non-owner call to `addPlugin` will revert.
  * In practice, the owner of the token contract and the owner of the plugin registry are not always the same entity.
  * This issue manifests immediately in live environments without requiring special conditions.

---

#### Proof of Concept

A Forge test (`PluginRegistryIsPluginAllowedPOC.t.sol`) demonstrates the issue:

```solidity
function testAddPluginRevertsForNonOwner() public {
    vm.prank(user);
    vm.expectRevert(
        abi.encodeWithSelector(
            Ownable.OwnableUnauthorizedAccount.selector,
            address(token)
        )
    );
    token.addPlugin(plugin);
}
```

The test confirms that any non-owner calling `addPlugin` reverts, even when the plugin is marked as allowed.

---

#### Recommendation

Remove the `onlyOwner` restriction from `isPluginAllowed` while preserving it on `updatePlugin`:

```diff
- function isPluginAllowed(address plugin) public view onlyOwner returns (bool) {
-     return allowedPlugins[plugin];
- }
+ /// @notice Returns true if the plugin is registered as allowed.
+ function isPluginAllowed(address plugin) public view returns (bool) {
+     return allowedPlugins[plugin];
+ }
```

This ensures that **any user can verify plugin allowance** and call `addPlugin`, while **only the owner retains authority to register or remove plugins**.


### [INFO-01] Unchecked `liquidationType` Enables Fake Events and Gas Griefing


#### Summary

The `AmmalgamPair::liquidate()` function does not validate its `liquidationType` parameter. Any value above 2 bypasses all liquidation logic yet still triggers a `Liquidate` event. This allows fake liquidation events and gas-griefing attacks, breaking the contract’s guarantee that liquidation events reflect real state changes.

---

#### Finding Description

The `liquidate` function is intended to support three valid liquidation types:

* **HARD (0)**
* **SOFT (1)**
* **LEVERAGE (2)**

However, because the implementation lacks a validation check, values `> 2` simply skip all logic but still emit a `Liquidate` event.

This creates two issues:

1. **Fake events**: Off-chain systems relying on logs may process or alert on liquidations that never occurred.
2. **Gas griefing**: Attackers can cheaply generate arbitrary liquidation events, bloating logs and wasting resources.

---

#### Impact Explanation

* **Off-chain confusion**: Risk systems, indexers, and bots may react to fake liquidation events, triggering false alarms or wasted actions.
* **Log pollution**: Repeated invalid calls clutter blockchain logs with meaningless data.
* **Gas griefing**: Attackers force unnecessary gas costs on infrastructure monitoring these events.

While no direct funds are at risk, the trustworthiness of liquidation event data is compromised.

---

#### Likelihood Explanation

* **High likelihood**: Any user with a borrow position can call `liquidate` with an invalid type.
* **No special conditions required**: Exploitation is trivial and possible at any time.

Thus, the likelihood of fake event emission is **high**, though the direct impact remains limited to off-chain systems.

---

#### Proof of Concept

A test demonstrates the vulnerability:

```solidity
function testLiquidateInvalidTypeEmitsEvent() public {
    uint256 debtXBefore = pair.tokens(BORROW_X).balanceOf(borrower);
    uint256 debtYBefore = pair.tokens(BORROW_Y).balanceOf(borrower);

    vm.startPrank(liquidator);
    vm.expectEmit(true, true, true, true);
    emit Liquidate(borrower, liquidator, 0, 0, 0, 0, 0, 0, 0, Liquidation.LEVERAGE + 1);
    pair.liquidate(borrower, liquidator, 0, 0, 0, 0, 0, 0, 0, Liquidation.LEVERAGE + 1);
    vm.stopPrank();

    assertEq(pair.tokens(BORROW_X).balanceOf(borrower), debtXBefore);
    assertEq(pair.tokens(BORROW_Y).balanceOf(borrower), debtYBefore);
}
```

This confirms that invalid liquidation types emit an event without altering debt balances.

---

#### Recommendation

Add validation to enforce valid types at the start of `liquidate`:

```solidity
function liquidate(/* … */, uint256 liquidationType) external lock {
    require(
        liquidationType <= uint256(Liquidation.LEVERAGE),
        "Invalid liquidation type"
    );
    accrueSaturationPenaltiesAndInterest(borrower);
    // …
}
```

This ensures that only legitimate liquidation types (0–2) are processed, preventing fake events and gas-griefing attacks.
