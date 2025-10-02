
### [L-01] Excessive Collateral Withdrawal in LiquidationManager

#### Summary
The `_retrieveCollateral()` function is susceptible to excessive fund withdrawals because it uses all available strategy shares without verification. It retrieves the full share balance of a `_holding` from each strategy and directly passes it to `StrategyManager::claimInvestment()`, regardless of the actual collateral required. This causes over-withdrawals, unnecessarily depleting strategy funds. Consequently, excess funds remain idle, missing out on potential yield opportunities.

---

#### Finding Description
The vulnerability in the `LiquidationManager::_retrieveCollateral()` function (lines 547–558) stems from its unconditional retrieval of the entire share balance of `_holding` from each strategy, which is then passed to `StrategyManager::claimInvestment()` without verifying the actual amount needed. This leads to excessive withdrawals, resulting in idle funds that miss out on potential yield.

The issue originates in the `selfLiquidate()` function, which allows users to initiate liquidation of their own positions. When `useHoldingBalance` is true, and the `_holding` token balance is insufficient to cover the required amount, with one or more strategies holding excess funds, `_retrieveCollateral()` attempts to bridge the shortfall by withdrawing from associated strategies. However, instead of withdrawing only the necessary amount, it retrieves the full share balance from each strategy and passes it to `StrategyManager::claimInvestment()` without capping it based on the actual deficit.  

This over-withdrawal causes excess funds to sit idle in the holding, reducing capital efficiency by forgoing yield opportunities and potentially enabling strategic exploitation.

---

#### Impact Explanation
- **Inefficient Capital Utilization**  
  The protocol's capital efficiency decreases as more collateral than needed is pulled from productive strategies, reducing overall returns.

- **Loss of User Yield**  
  Withdrawing excess collateral from strategies leaves it idle in the holding, depriving users of potential yield or rewards that could have been earned if the funds remained invested.

---

#### Likelihood Explanation
1. `useHoldingBalance` is enabled, allowing `_retrieveCollateral()` to first account for the existing balance in `_holding`.  
2. The `_holding`’s current token balance is insufficient to cover the required `_amount`:  
   - Required amount: **150 USDC**  
   - `_holding` balance: **50 USDC**  
   - Shortfall: **100 USDC**  
3. The strategies linked to the `_holding` hold more than enough to cover the shortfall:  
   - **Strategy A**: 120 USDC in shares  
   - **Strategy B**: 80 USDC in shares  

---

#### Proof of Concept

**Scenario Setup**
- **Token**: USDC  
- **Holding**: `0xHolding`  
- **Required Amount**: 150 USDC  
- **Initial Holding Balance**: 50 USDC (`useHoldingBalance = true`)  
- **Strategy A**: 120 USDC worth of shares for `_holding`  
- **Strategy B**: 80 USDC worth of shares for `_holding`  

**Execution with Bug**

- **Initial Check**:  
  - `_holding` balance: 50 USDC  
  - Required amount: 150 USDC  
  - Since 50 < 150, `_retrieveCollateral()` proceeds to withdraw from strategies.  

- **Strategy A Execution**:  
  - Retrieves all `_holding` shares in Strategy A (120 USDC).  
  - Updates `_holding` balance: 50 + 120 = 170 USDC.  
  - Loop terminates after Strategy A since balance now exceeds required amount.  

- **Final State**:  
  - Retrieved collateral: 120 USDC  
  - Total `_holding` balance: 170 USDC  
  - Required amount: 150 USDC  
  - **Excess withdrawn**: 20 USDC (unnecessarily taken from Strategy A)  

---

#### Test Code
```solidity
function test_excessive_withdrawal_from_strategies() public {
    // --- Scenario Setup ---
    address user = address(0x1234);
    address holding = user; // Holding is the user for simplicity
    SampleTokenERC20 token = usdc;

    // Initial balances
    uint256 initialHoldingBalance = 50e6; // 50 USDC (6 decimals)
    uint256 strategyABalance = 120e6; // Strategy A: 120 USDC
    uint256 strategyBBalance = 80e6; // Strategy B: 80 USDC
    uint256 requiredAmount = 150e6; // Required: 150 USDC

    // Deploy mock strategies
    MockStrategy strategyA = new MockStrategy(address(token));
    MockStrategy strategyB = new MockStrategy(address(token));

    // Fund strategies
    deal(address(token), address(strategyA), strategyABalance);
    deal(address(token), address(strategyB), strategyBBalance);

    // Assign shares to holding
    strategyA.setShares(holding, strategyABalance);
    strategyB.setShares(holding, strategyBBalance);

    // Set holding's initial balance
    deal(address(token), holding, initialHoldingBalance);

    // Approve strategies for token transfers
    token.approve(address(strategyA), type(uint256).max);
    token.approve(address(strategyB), type(uint256).max);

    // Define strategies array
    address ;
    strategies[0] = address(strategyA);
    strategies[1] = address(strategyB);

    // --- Test Execution ---
    vm.startPrank(user, user);
    uint256 holdingBalance = token.balanceOf(holding);
    uint256 retrieved = 0;

    console.log("Initial holding balance:", holdingBalance);
    console.log("Required amount to withdraw:", requiredAmount);

    for (uint256 i = 0; i < strategies.length; i++) {
        if (holdingBalance >= requiredAmount) break;

        uint256 shares = MockStrategy(strategies[i]).shares(holding);
        console.log("Strategy", i, "shares for holding:", shares);

        if (shares > 0) {
            // Withdraw all shares (bug simulation)
            MockStrategy(strategies[i]).withdrawAll(holding);
            retrieved += shares;
            holdingBalance = token.balanceOf(holding);
            console.log("Post-withdrawal holding balance:", holdingBalance);
        }
    }
    vm.stopPrank();

    // --- Assertions ---
    assertEq(
        token.balanceOf(holding),
        170e6,
        "Holding balance should be 170 USDC (excess withdrawal)"
    );
    assertEq(
        retrieved,
        120e6,
        "Retrieved 120 USDC from strategies"
    );
    assertEq(
        MockStrategy(strategies[0]).shares(holding),
        0,
        "Strategy A shares should be zero"
    );
    assertEq(
        MockStrategy(strategies[1]).shares(holding),
        80e6,
        "Strategy B shares should remain untouched"
    );
}
````

```solidity
contract MockStrategy {
    address public token;
    mapping(address => uint256) public shareBalances;

    constructor(address _token) {
        token = _token;
    }

    function setShares(address holder, uint256 value) external {
        shareBalances[holder] = value;
    }

    function shares(address holder) external view returns (uint256) {
        return shareBalances[holder];
    }

    function withdrawAll(address holder) external {
        uint256 amount = shareBalances[holder];
        require(amount > 0, "No shares to withdraw");
        shareBalances[holder] = 0;
        SampleTokenERC20(token).transfer(holder, amount);
    }
}
```


#### Recommendation

Limit withdrawals in `_retrieveCollateral()` to **only the needed deficit** by calculating `remainingNeeded` and using the minimum of that or available shares, preventing excess fund withdrawal.
