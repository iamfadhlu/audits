# CLEMENTINE
[Contest Details](https://cantina.xyz/code/ce181972-2b40-4047-8ee9-89ec43527686)

## [Info-01] Broadcast-before-persist crash window

### Summary
The Clementine bridge's transaction sender exhibits a classic broadcast-before-persist race condition where transactions are broadcast to the Bitcoin network before database state is committed. If the system crashes between broadcasting and persisting state, the same transaction may be rebroadcast on restart, potentially causing duplicate payouts and fund loss.

### Finding Description
The vulnerability exists in the transaction sending pipeline where network broadcast occurs before database persistence is committed. In the send_no_funding_tx() function, the system calls self.rpc.send_raw_transaction(&tx) to broadcast the transaction to the Bitcoin network, then updates debug state in the database afterward.

The transaction sending workflow in try_to_send_unconfirmed_txs() processes transactions from the database queue and calls the appropriate sending method (CPFP, RBF, or no-funding) for each transaction. However, there's no transactional coordination between the network broadcast and database state updates.

The system lacks several critical safeguards: there's no pre-broadcast persistence of idempotency keys, no unique constraints preventing duplicate transaction processing, and no restart reconciliation logic to detect already-broadcast transactions. The transaction sender operates with inline broadcasting rather than using a transactional outbox pattern that would ensure exactly-once semantics.

### Impact Explanation
A crash occurring between transaction broadcast and database commit could result in duplicate transaction broadcasts on system restart. This is particularly dangerous for withdrawal processing where operators broadcast payout transactions - a duplicate payout could drain operator collateral and cause financial losses. The vulnerability affects the integrity of the bridge's transaction processing and could be exploited to extract funds through repeated crashes during the critical broadcast-persist window.

### Likelihood Explanation
The vulnerability requires precise timing where a system crash occurs in the narrow window between network broadcast and database commit. While this timing window is small, the high-value nature of bridge transactions and the potential for systematic exploitation through repeated crashes increases the practical risk. The likelihood is elevated in distributed environments where multiple operators process transactions independently and system restarts are common during maintenance or failures.

### Recommendation
Consider implementing a transactional outbox pattern that persists transaction intent before network broadcast. One approach could be to modify the transaction sending flow to use pre-broadcast state persistence:

```diff
pub(super) async fn send_no_funding_tx(
    &self,
    try_to_send_id: u32,
    tx: Transaction,
    tx_metadata: Option<TxMetadata>,
) -> Result<()> {
+   // Persist broadcast intent before network operation
+   let mut dbtx = self.db.begin_transaction().await?;
+   self.db.mark_transaction_broadcasting(&mut dbtx, try_to_send_id).await?;
+   dbtx.commit().await?;
+   
    match self.rpc.send_raw_transaction(&tx).await {
        Ok(sent_txid) => {
            // Update success state
        }
        Err(e) => {
            // Handle broadcast failure
        }
    }
}
```

Additionally, consider implementing startup reconciliation logic that compares database state against the blockchain to detect already-broadcast transactions, and add comprehensive integration tests that simulate crash scenarios during the broadcast-persist window.


## [Info-02] Timelock can be 0 blocks#289

### Summary
The Clementine bridge's timelock implementation allows zero-block timelock values that provide no actual delay protection for user deposits. The TimelockScript constructor and generate_deposit_address function accept user_takes_after: u16 values of zero without validation, creating OP_CSV 0 scripts that can be spent immediately, bypassing the intended user recovery mechanism. Severity: Medium

### Finding Description
The vulnerability exists in the timelock validation logic across multiple components. The generate_deposit_address() function in address.rs:148-171 accepts a user_takes_after: u16 parameter without any lower-bound validation, directly passing it to TimelockScript::new().

The TimelockScript::new() constructor in script.rs:419-422 accepts any block_count: u16 value without validation. When the script is generated via to_script_buf(), it creates a Bitcoin script using script.rs:394-408 that pushes the block count value and uses OP_CSV. A zero value results in OP_CSV 0, which provides no delay constraint.

The timelock script is used in deposit addresses where it's intended to provide users a recovery window if deposits fail. In the deposit system, this timelock is created as part of the Taproot script tree alongside the main deposit script, allowing users to reclaim funds after a specified delay.

### Impact Explanation
A zero-block timelock completely eliminates the intended user protection mechanism, allowing immediate spending of deposit funds through the recovery path without any delay. This undermines the security model where users should have a grace period to recover funds if the bridge fails to process their deposit properly. Users could potentially exploit this to bypass normal deposit processing or attackers could immediately drain deposits that should be time-locked.

### Likelihood Explanation
The vulnerability requires a zero value to be passed as the user_takes_after parameter during deposit address generation. This could occur through configuration errors, API misuse, or malicious input. The CLI implementation shows default values being used, but the system accepts any value without validation, making the issue exploitable through direct API calls or configuration manipulation.

### Recommendation
Consider implementing comprehensive validation to enforce minimum timelock values across all creation paths. The most effective approach would be to add validation in both the TimelockScript::new() constructor and generate_deposit_address() function:

```diff
pub fn new(xonly_pk: Option<XOnlyPublicKey>, block_count: u16) -> Self {
+   if block_count == 0 {
+       panic!("Timelock block count must be at least 1");
+   }
    Self(xonly_pk, block_count)
}
```
Additionally, consider adding validation in generate_deposit_address() before creating the timelock script, and implement comprehensive test coverage that explicitly verifies zero-timelock scenarios are rejected during construction.

Notes
The timelock system is used throughout the bridge protocol in various transaction types operator_reimburse.rs:143-146 and operator_collateral.rs:99-100.

## [Info-02] RBF fee bump underestimates relay requirements#280

### Summary
The Clementine bridge's RBF fee calculation uses a hardcoded 222 vbyte constant that may underestimate Bitcoin network relay requirements. This approach fails to account for dynamic mempool conditions, package replacements, and actual node policies, potentially causing transaction broadcast failures and degraded bridge liveness.

### Finding Description
The calculate_bump_feerate() function in the transaction sender implements RBF fee calculation using a fixed increment approach. The implementation calculates the minimum bump fee rate as original_feerate + (222f64 / original_tx_vsize as f64), assuming a constant 222 vbyte increment will satisfy Bitcoin's incremental relay requirements.

This calculation method exhibits several technical limitations. The function retrieves only the original transaction for analysis and uses its vsize throughout the calculation, without considering that replacement transactions may have different sizes. The implementation also lacks integration with live node policy queries and does not account for package replacement scenarios where multiple transactions may be evicted.

The RBF logic is invoked through send_rbf_tx() when bumping existing transactions, which then uses the calculated fee rate for psbt_bump_fee operations. This affects the bridge's ability to reliably broadcast transactions during periods of network congestion or when dealing with larger transaction sizes.

### Impact Explanation
Transaction broadcast failures may occur when the calculated fee increment proves insufficient for actual network conditions. This can result in bridge transactions remaining unconfirmed for extended periods, degrading the user experience as deposits and withdrawals appear "pending forever." The impact primarily affects bridge liveness rather than fund security, as transactions remain valid but may require manual intervention or alternative fee-bumping strategies.

### Likelihood Explanation
The vulnerability manifests under specific network conditions including mempool congestion, large transaction sizes, or scenarios requiring package replacements. While the fixed 222 vbyte heuristic works for simple cases, it becomes inadequate when Bitcoin's dynamic relay policies require higher increments. The likelihood increases during periods of high network activity when precise fee calculation becomes more important for transaction propagation.

### Recommendation
Consider implementing dynamic fee calculation that queries live node policy through getmempoolinfo to obtain current incremental relay requirements. One approach could be to replace the hardcoded constant with a function that calculates the required absolute fee increase based on the sum of replaced transaction fees plus the network's current minimum increment.

```diff
- let min_bump_feerate = original_feerate + (222f64 / original_tx_vsize as f64);
+ let network_increment = self.get_network_min_increment().await?;
+ let min_bump_feerate = original_feerate + (network_increment / original_tx_vsize as f64);
```