## AAVE APTOS
[Contest Details](https://cantina.xyz/competitions/ad445d42-9d39-4bcf-becb-0c6c8689b767)

### [M-01] Stale Oracle Price Liquidation

#### Summary
A vulnerability in `liquidation_logic.move` allows attackers to exploit stale oracle prices during market volatility, enabling excessive collateral extraction through self-liquidation, resulting in significant protocol losses.

#### Finding Description
The `calculate_available_collateral_to_liquidate` function in `liquidation_logic.move` (lines 655-656) fetches asset prices using `oracle::get_asset_price` without validating the freshness of the price data. This lack of timestamp validation allows attackers to exploit discrepancies between stale collateral prices and fresh debt prices during volatile market conditions. Specifically, a stale high collateral price combined with a fresh low debt price inflates the liquidation bonus, enabling attackers to extract excess collateral.

##### Security Guarantees Broken
- Integrity of Liquidation Mechanism: The liquidation process relies on accurate pricing to ensure fair collateral distribution. Stale prices undermine this, allowing attackers to manipulate the system for profit.
- Economic Security: The protocol's collateral reserves are at risk, as attackers can extract more collateral than intended, leading to financial losses.
##### Attack Propagation
- Setup: An attacker borrows a stablecoin (e.g., USDC) using a volatile asset (e.g., BTC) as collateral.
- Market Volatility: The attacker monitors the market for a price crash in the collateral asset (BTC), where the oracle price lags behind the real-time market price.
- Self-Liquidation: The attacker triggers a self-liquidation when the oracle still reports a stale, high BTC price, while the debt asset (USDC) has a fresh price.
- Profit Extraction: The liquidation bonus is calculated using the stale high BTC price, allowing the attacker to receive excess collateral, which can be sold at the current market price for profit.
This vulnerability is exploitable because the `oracle::get_asset_price` function does not check the timestamp of the price data, despite the availability of `get_benchmark_timestamp` functionality in the Chainlink infrastructure, indicating that timestamp validation is feasible but not implemented.

#### Impact Explanation
Attackers can extract excess collateral, directly reducing the protocol's reserves. The liquidation mechanism is a critical component of the protocol, and its compromise undermines user trust and protocol stability. During volatile market conditions, the difference between stale and fresh prices can be substantial, leading to large-scale collateral losses.

The impact is severe because it affects a core economic function of the protocol and can be exploited without requiring complex conditions beyond market volatility, which is a common occurrence.

#### Likelihood Explanation
Price crashes or rapid fluctuations in assets like BTC are frequent, creating opportunities for exploitation.

The combination of a practical attack vector and the lack of basic validation makes this issue highly likely to be exploited in real-world scenarios.

#### Recommendation
To mitigate this vulnerability, implement a staleness check for oracle prices in the `calculate_available_collateral_to_liquidate` function. The check should ensure that the price data is recent by comparing the oracle's last update timestamp with the current time.


### [L-01] Aave Isolation Mode Debt Ceiling Bypass Vulnerability

#### Summary
The Aave protocol's isolation mode contains a vulnerability where attackers can bypass debt ceilings by borrowing small amounts that are truncated to zero due to precision loss, risking protocol insolvency.

#### Finding Description
In `borrow_logic.move` module, specifically within the `internal_borrow` function (lines 205-207), the borrowed amount is scaled down before being added to the `isolation_mode_total_debt`. The scaling factor is calculated as `decimals = reserve_decimals - 2` (lines 201-203), where `reserve_decimals` is the number of decimals of the underlying asset (e.g., 18 for USDC), and 2 is a constant from `reserve_config.move:77`. This results in a scaling factor of 10^16 for USDC.

The vulnerable code is:
```rust
let next_isolation_mode_total_debt = 
    (isolation_mode_total_debt as u256) + 
    (amount / math_utils::pow(10, decimals));
pool::set_reserve_isolation_mode_total_debt(...);
```
When the borrowed amount is less than 10^decimals (e.g., 10^16 wei or 0.01 USDC), the integer division amount / math_utils::pow(10, decimals) truncates to 0. This means small borrows do not increase the tracked `isolation_mode_total_debt`, allowing an attacker to accumulate actual debt without breaching the debt ceiling.

##### Security Guarantees Broken
- Debt Ceiling Enforcement: The protocol assumes that the debt ceiling strictly limits borrowing in isolation mode. However, precision loss allows debt to accumulate invisibly, breaking this guarantee.
- Risk Management: Isolation mode is designed to mitigate risk by capping debt against specific collateral. This vulnerability undermines that by permitting unchecked debt growth.

###### Propagation of Malicious Input
- An attacker targets an isolated asset like USDC (18 decimals).
- They repeatedly borrow small amounts, e.g., 0.009 USDC (9 × 10^15 wei).
- For each borrow, the scaled debt addition (9 × 10^15) / 10^16 truncates to 0.
- The `isolation_mode_total_debt` remains unchanged, while the actual debt increases via `variable_debt_token_factory::mint`.
- The protocol continues to allow borrows since the debt ceiling appears unbreached.
- The accumulated untracked debt can exceed the collateral’s value, leading to potential insolvency.
- This pattern is systemic, as similar scaling logic appears in `isolation_mode_logic.move:47-48` for repayments, reinforcing the vulnerability’s consistency across the codebase.

#### Impact Explanation
The ability to bypass debt ceilings poses a severe risk of protocol insolvency. If exploited at scale, attackers could accumulate debt far beyond intended limits, exhausting reserves or leaving the protocol with unrecoverable bad debt, especially if collateral values drop. The high impact stems from the direct threat to the protocol’s financial stability and user trust.

#### Likelihood Explanation
Exploiting this requires executing numerous small transactions, which increases gas costs and visibility on-chain. However, it is technically feasible and could be automated via a script or smart contract. The subtlety of the issue—relying on precision loss rather than obvious flaws—may delay detection, making it moderately likely to occur if discovered by a motivated attacker.

#### Recommendation
To fix this, enforce a minimum borrow amount that ensures the scaled debt is at least 1 unit. Add a check before updating the debt:

```rust
let debt_units = amount / math_utils::pow(10, decimals);
assert!(debt_units > 0, "Borrow amount too small to track against debt ceiling");
let next_isolation_mode_total_debt = 
    (isolation_mode_total_debt as u256) + debt_units;
```

This prevents borrows that truncate to zero, ensuring all debt is tracked. Additionally, consider revising the scaling logic to use higher-precision arithmetic or adjust the debt ceiling’s decimal precision to minimize truncation risks.


### [L-02] No Mechanism to Handle Bad Debt Leading To Insolvency

#### Summary
The Aave protocol lacks an automated mechanism to manage bad debt, as identified in `liquidation_logic.move`. There is no process to handle or mitigate these deficits, potentially leading to protocol insolvency.

#### Finding Description
The Aave protocol assumes that liquidations will always fully cover undercollateralized debt, as implied by the logic in `liquidation_logic.move`. However, this assumption fails in volatile markets where the collateral's value may drop significantly below the borrowed amount. When a liquidation occurs, the `DeficitCreated` event is emitted to signal a shortfall (i.e., bad debt) when the liquidated collateral does not cover the outstanding debt. However, the protocol does not include an automated mechanism to address this deficit, such as reserve funds, protocol fees allocation, or bad debt redistribution.

This breaks the core invariant of protocol solvency, as unhandled bad debt can accumulate, eroding the protocol's ability to cover lenders' assets.

###### Example scenario
- A user supplies volatile collateral (e.g., a low-liquidity token) and borrows a stable asset (e.g., USDC).
- The price drops and the position somehow slips getting liquidated due to the position being too large to be covered by liquidation or not enough incentive due to the position being small
- The deficit accumulates, impacting the protocol's reserves and potentially leading to insolvency.

This issue propagates through the system as follows:
- In `liquidation_logic.move`, the liquidation process calculates the collateral to be seized and the debt to be repaid.
- If the collateral value is less than the debt, a deficit is recorded via `DeficitCreated`.
- Without a mechanism to handle this deficit, it remains unresolved, accumulating across multiple liquidations.


#### Impact Explanation
Undercollateralization of the protocol and Insolvency

#### Likelihood Explanation
It depends on market conditions, specifically the volatility of collateral assets.

#### Recommendation
To address this issue, the protocol should implement an bad debt management mechanism such as bad debt redistribution