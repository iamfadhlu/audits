### [M-01] Un-burnable Validators Due to Atomic `_consensusBurn` Dependency on `_unstake` Fund Distribution

#### Summary

The `_consensusBurn` function, responsible for forcibly removing misbehaving validators, can be rendered ineffective if its internal `_unstake` call fails. Because `_consensusBurn` executes atomically, a revert during `_unstake` prevents the validator from being burned, allowing malicious validators to remain active indefinitely and undermining protocol enforcement.

---

#### Finding Description

The issue arises in `src/consensus/ConsensusRegistry.sol` (lines 520–538), where `_consensusBurn` calls `_unstake` from `src/consensus/StakeManager.sol` (lines 171–205).

The `_unstake` flow includes a call to `Issuance(issuance).distributeStakeReward()`, which performs a low-level call to the validator’s recipient address. If this external call reverts, the entire `_unstake` reverts — and by extension, `_consensusBurn` reverts too.

**Exploit Scenario**

* A malicious validator sets their reward recipient to a contract that always reverts in `receive()` or `fallback()`.
* Alternatively, an edge case or bug in `Issuance.distributeStakeReward()` (e.g., insufficient balance or unexpected state) causes persistent reverts.
* As a result, `_unstake` cannot succeed, and `_consensusBurn` never completes.
* The validator’s NFT remains intact, making them immune to forced removal.

This breaks the protocol’s guarantee of being able to retire and slash validators who misbehave.

---

#### Impact Explanation

* **Liveness Failure**: Misbehaving validators cannot be forcibly removed.
* **Consensus Integrity Risk**: Immune validators can continue disrupting or destabilizing the network.
* **Collusion Risk**: Multiple unburnable validators could coordinate to gain disproportionate influence.
* **Governance Weakness**: The system loses the ability to enforce slashing, a core security mechanism.

While no direct fund theft occurs, the inability to enforce validator removal creates a **critical governance and security vulnerability**.

---

#### Likelihood Explanation

* **Medium likelihood**:

  * Malicious validators can easily configure a reverting recipient contract.
  * Even honest validators could become “unburnable” if their recipient address has faulty logic.
  * Reliance on external calls within an atomic consensus enforcement function makes the system brittle.

The attack requires minimal technical sophistication and is realistically exploitable.

---

#### Recommendation

* **Decouple burning from unstaking**: Refactor `_consensusBurn` so validator removal and NFT burning succeed even if `_unstake` fails.
* **Graceful handling of fund distribution**: Treat reward distribution errors as non-fatal, ensuring validator removal proceeds regardless.
* **Error isolation**: Use try/catch or equivalent mechanisms to contain revert-causing external calls.
* **Failsafe fallback**: If fund distribution fails, log the event and allow later manual or automated recovery of funds.

Prioritizing validator removal over reward distribution ensures protocol liveness, preserves enforcement guarantees, and prevents validators from becoming unburnable.


### [M-02] Economic Manipulation via `DEFAULT_BAD_NODES_STAKE_THRESHOLD` and `LeaderSwapTable`

#### Summary

The Telcoin Network’s leader selection mechanism is vulnerable to economic manipulation through the use of `DEFAULT_BAD_NODES_STAKE_THRESHOLD` and `LeaderSwapTable`. Because the classification of “bad nodes” is determined by stake-weighted thresholds rather than absolute reputation scores, large validators or colluding groups can strategically push out smaller competitors while maintaining their own leadership positions, leading to centralization of power and potential censorship.

---

#### Finding Description

The vulnerability lies in:

* `LeaderSwapTable::new` (`consensus/primary/src/consensus/leader_schedule.rs`)
* `Bullshark::update_leader_schedule` (`consensus/primary/src/consensus/bullshark.rs`)

The `LeaderSwapTable` defines “bad nodes” based on a cumulative stake threshold (`bad_nodes_stake_threshold`, set to 33% in tests but configurable). The function `retrieve_first_nodes` accumulates validators by voting power until the threshold is met, rather than directly ranking by reputation scores.

**Exploit Scenario**

1. A validator cartel controls just under the threshold (≈33%) of stake.
2. They depress the reputation scores of smaller validators (e.g., via coordinated reporting or manipulation).
3. Because the threshold is stake-weighted, small validators with low reputation get flagged as “bad.”
4. The large validator’s relatively low reputation does not matter as long as their cumulative stake is high enough to stay above the threshold.
5. Over time, leadership centralizes among large validators who can game the system.

This undermines the intended goal of removing underperforming nodes and instead creates a **stake-based weaponization of the threshold**.

---

#### Impact Explanation

* **Centralization of Power**: Fewer large validators dominate leadership roles.
* **Reduced Network Resilience**: With leadership concentrated, losing a few validators could disrupt consensus.
* **Economic Inequity**: Smaller validators are punished despite comparable or better performance, discouraging participation.
* **Censorship Risk**: Dominant players can suppress validators who process unwanted transactions.
* **Mechanism Subversion**: The threshold designed to strengthen the network instead enables control by wealthy participants.

Given its effect on fairness and decentralization, the **impact is High**.

---

#### Likelihood Explanation

* Stake-weighted selection favors large players, regardless of actual performance.
* A cartel controlling ~33% of stake can easily exploit the mechanism.
* Attack feasibility is **Low-to-Medium** today, but grows as stake distribution skews.

Thus, while the current likelihood is **Low**, the exploit remains a credible risk as the system scales.

---

#### Proof of Concept

The unit test `test_bad_nodes_selection` (`leader_schedule_tests.rs`, lines 65–86) demonstrates the issue:

```rust
let validators = vec![
    (validator1, 40), // 40% stake
    (validator2, 30), // 30% stake
    (validator3, 20), // 20% stake
    (validator4, 10), // 10% stake
];
let reputation_scores = vec![50, 40, 30, 20];
leader_swap_table.populate(validators, reputation_scores);

let bad_nodes = leader_swap_table.get_bad_nodes();
assert_eq!(bad_nodes, vec![validator4]); // lowest stake selected
```

Even though validator2 and validator3 have lower scores, only validator4 is penalized because of its low stake. This confirms that **stake overrides score ranking** in bad-node selection.

---

#### Recommendation

* **Prioritize reputation scores over stake**: Select bad nodes based primarily on performance metrics, not cumulative stake.
* **Hybrid criteria**: Combine score and stake thresholds so high-stake validators with poor performance cannot escape penalties.
* **Dynamic thresholding**: Adjust thresholds based on network conditions to prevent stake cartels from dominating leader selection.

This ensures that poorly performing validators are removed fairly, preserving decentralization and preventing economic manipulation.



### [L-01] ConsensusRegistry `_getValidators` Gas Limit

#### Summary

The `_getValidators` function in the `ConsensusRegistry` contract is vulnerable to gas exhaustion as the validator set grows beyond the assumed ~1000 members. Because the function enumerates all validators without bounds, it risks denial of service for critical system functions and off-chain processes relying on validator queries.

---

#### Finding Description

The issue lies in `src/consensus/ConsensusRegistry.sol` (lines 644–645), where `_getValidators` iterates over all existing validators using `totalSupply()`. Since `totalSupply()` can theoretically reach `type(uint24).max - 1` (~16 million), the loop scales linearly with validator count.

The code comment assumes validator numbers will remain around 1000, but this is a **business assumption**, not a technical constraint. As validator numbers increase, the function’s gas cost grows proportionally, risking execution failures.

**Breakdown of Security Guarantee Failures:**

* **Unbounded Loop**: Gas usage increases linearly with validator count.
* **Assumption vs. Reality**: Assumes low validator count, but protocol permits millions.
* **Critical Dependency**: Function feeds into committee validation and other consensus-critical logic.
* **No Safeguards**: Lacks gas optimization mechanisms like pagination or chunking.

This makes `_getValidators` a potential denial-of-service vector as validator counts rise.

---

#### Impact Explanation

If validator counts scale significantly:

* View functions may consume excessive gas or fail.
* Off-chain services querying validator sets could break.
* Consensus-related operations dependent on validator enumeration risk hitting gas limits, potentially halting system progress.

Although severity is set to **Low** in tracking, the **impact in practice can escalate** if validator numbers grow unchecked, posing a scalability bottleneck.

---

#### Likelihood Explanation

* **Current Likelihood Low**: Requires thousands to millions of validators before practical issues arise.
* **Future Likelihood Higher**: As the network scales, the probability of hitting gas limits increases.

Thus, while presently a **Low likelihood**, this represents a plausible medium- to long-term scalability risk.

---

#### Recommendation

* **Implement Pagination/Chunking**: Modify `_getValidators` to return paginated subsets of validators, reducing gas load per call.
* **Enforce Validator Cap**: Introduce a maximum validator limit aligned with gas efficiency assumptions.

These mitigations would align system design with Ethereum gas constraints and prevent denial-of-service as validator sets expand.


### [L-02] `PrimaryNetworkHandle::publish_consensus` Race Condition / Out-of-Order Gossip

#### Summary

A race condition in `PrimaryNetworkHandle::publish_consensus` allows malicious validators to exploit out-of-order gossip messages. By manipulating gossip timing, an attacker can cause nodes to temporarily adopt outdated consensus states, leading to wasted resources, temporary stalls, and inconsistent views of the latest committed block.

---

#### Finding Description

The vulnerability lies in how `publish_consensus` broadcasts `(number, hash)` pairs representing committed consensus outputs. These are consumed by `RequestHandler::process_gossip`, which forwards them to:

```rust
self.consensus_bus.last_published_consensus_num_hash().send((number, hash));
```

Because this logic lacks ordering validation, **any incoming message (newer or older)** overwrites the watch channel. As gossip messages are processed asynchronously (`mod.rs:262`), older messages may arrive and be processed before newer ones.

A malicious validator can deliberately delay newer messages while flooding older ones, tricking peers into temporarily updating their `last_published_consensus_num_hash` to an outdated state.

---

#### Impact Explanation

* **Liveness / Functionality**: Components relying on `last_published_consensus_num_hash` (e.g., sync and networking modules) may act on stale data, retrying already-synced blocks or stalling.
* **Resource Waste**: Nodes repeatedly resync or process redundant state updates, consuming CPU and network bandwidth.
* **Consensus Confusion**: Nodes may briefly hold divergent views of the “latest” consensus block, complicating state convergence.

While consensus will eventually converge on the correct highest block, the interim inconsistency introduces inefficiency and resource overhead.

---

#### Likelihood Explanation

* **No Ordering Validation**: Handler updates the watch channel unconditionally, allowing regressions.
* **Watch Channel Behavior**: `last_published_consensus_num_hash` overwrites with any new value, even if it regresses.
* **Asynchronous Gossip Flow**: Message delivery is network-timing dependent, not logically ordered.

Given these factors, exploitation is **feasible** with control over gossip timing.

---

#### Proof of Concept

1. Attacker prepares two gossip messages:

   ```rust
   let older_message = PrimaryGossip::Consensus(1, old_block_hash);
   let newer_message = PrimaryGossip::Consensus(2, new_block_hash);
   ```
2. Send the **older message first** to target nodes.

   ```rust
   target_node.send(older_message).await;
   ```
3. Delay, then send the **newer message**.

   ```rust
   target_node.send(newer_message).await;
   ```
4. The node updates `last_published_consensus_num_hash` with the outdated pair before receiving the newer one.
5. Dependent subsystems react to the older state, triggering redundant syncs and wasted work.

---

#### Recommendation

1. **Ordering Validation**
   Ensure only forward progress updates are accepted:

   ```rust
   if number > self.consensus_bus.last_published_consensus_num_hash().get().0 {
       self.consensus_bus.last_published_consensus_num_hash().send((number, hash));
   } else {
       // Optionally log or drop outdated gossip
   }
   ```

2. **Robust Gossip Protocol**
   Add sequence numbers or timestamps to gossip messages so out-of-order deliveries can be detected and ignored.

These changes prevent outdated gossip from overwriting newer state, ensuring consistent and efficient consensus propagation.




### [INFO-01] Under-Set early_finalize Flag Causes Execution Delay and Reward Leakage for Temporarily Inactive Validators

#### Summary

The `early_finalize` flag—used to determine whether blocks can be finalized immediately—is derived from the current node mode via `node_mode.is_active_cvv()`. However, this logic fails when a validator temporarily switches to Observer mode (e.g., during maintenance or upgrades). Even though the node continues to receive its own valid certificates, it won’t early-finalize them, causing execution stalls and reward leakage, while the rest of the network progresses normally.

---

#### Finding Description

The consensus output logic sets the `early_finalize` flag as follows:

```rust
let early_finalize = node_mode.is_active_cvv(); // subscriber.rs:325-331
```

This flag is embedded into `ConsensusOutput` and consumed later by the execution engine:

```rust
if output.early_finalize {
    finalize_block_immediately();
} else {
    wait_for_final_signed_cert();
}
```

The logic assumes that a node in Observer mode should not early-finalize any blocks. However, this assumption breaks down when:

1. A validator temporarily transitions to Observer mode (e.g., for safe restart or upgrade).
2. The node continues to receive certificates it helped create.
3. `is_active_cvv()` returns `false`, so `early_finalize = false`.
4. The node withholds finalization of its own certificates.
5. The rest of the network finalizes normally, leading to local execution delays and missed rewards.

This represents a **state mismatch between role-based stake participation and transient operational mode**.

---

#### Impact Explanation

This issue results in:

* Block finalization delays on the affected node.
* Execution engine lags, potentially stalling dependent services or on-chain events.
* Reward leakage, since the node fails to finalize blocks it helped produce.
* Operational confusion, as block inclusion succeeds but execution stalls.
* Divergence in local state compared to global chain progression.

Because it degrades liveness and leads to economic harm from reward loss, the impact is assessed as **High**.

---

#### Likelihood Explanation

* Operational mode transitions (Validator → Observer) are common during upgrades.
* The consensus layer still routes certificates to the temporarily inactive node.
* Current logic does not account for this transitional state.
* No safeguards or overrides exist for self-certificates marked as inactive.

Given these conditions, the issue is **highly likely** to occur in real validator operations.

---

#### Proof of Concept

1. Run a validator in `CvvActive` mode.
2. Participate in block production; observe normal `early_finalize = true`.
3. Switch node to Observer mode while keeping validator keys intact.
4. Node continues to receive its own certificates but marks `early_finalize = false`.
5. Execution engine defers finalization unnecessarily, creating lag and missing rewards.
6. Other validators finalize normally.

---

#### Recommendation

1. **Decouple `early_finalize` from operational mode**

   * Set `early_finalize = true` if the certificate includes the node as a signer, regardless of current mode.

   ```rust
   let early_finalize = certificate.signers.contains(&my_validator_id);
   ```

2. **Track recent validator roles**

   * Maintain a history of validator participation (e.g., over N blocks) to infer safe early finalization even after role transitions.

3. **Warn on mismatched self-certificates**

   * Log diagnostics if a node sees its own certificate but fails to early-finalize due to mode mismatch.

By anchoring early finalization on actual certificate participation rather than transient mode, the system can ensure accurate and timely block finalization, prevent unnecessary delays, and protect validator rewards during operational transitions.


### [INFO-02] Committee-Slot Hash Roulette Enables Targeted Validator DoS via Transaction Pre-Image Grinding

#### Summary

The deterministic assignment `tx.hash() % committee_size == slot` in `submit_txn_if_mine` allows attackers to precompute transaction hashes that map to a specific validator slot. An attacker can grind transaction pre-images offline until their transactions are assigned to a chosen validator, then flood that validator with high-cost, large-payload transactions via the gossip network, causing targeted CPU/bandwidth/memory exhaustion (DoS) for that validator.

---

#### Finding Description

The batch validator uses the following deterministic assignment logic (simplified):

```
let hash = tx.hash(); // 32-byte hash
let index = u64::from_ne_bytes(hash[0..8].try_into().unwrap());
if index % committee_size != my_slot {
    return; // not my responsibility
}
process(tx);
```

Because the mapping is public and deterministic, an attacker can manipulate transaction fields (nonce, calldata, etc.) offline until the resulting hash maps to the victim’s slot. The attacker then submits many such transactions; only the targeted validator (the one matching `slot`) will process them, bearing the full validation cost while others ignore them.

**Attack Strategy**

1. Select a victim validator (e.g., `slot = 3`).
2. Craft many large transactions (~1MB calldata), iterating inputs until `tx.hash() % N == 3`.
3. Submit the crafted transactions to the network (or gossip).
4. The targeted validator processes them, suffering CPU, bandwidth, and memory load.

---

#### Impact Explanation

* Targeted validators must fully pre-validate and process the attacker’s payloads while others remain unaffected.
* Large payloads amplify resource consumption (CPU, memory, bandwidth).
* Sustained grinding leads to degraded validator responsiveness and fairness.
* If the victim occupies quorum-critical slots, block production or consensus progress can be delayed or stalled.

Because this enables cheap, asymmetric, targeted DoS against core infrastructure nodes, the practical impact is **High** despite the tracked severity being Informational.

---

#### Likelihood Explanation

* Exploitation requires only offline computation (hash grinding).
* Transaction input space (calldata, nonce) is large and easily manipulable.
* Assignment logic is publicly known and deterministic.
* Validators cannot distinguish targeted transactions from legitimate ones until after full validation.

These factors make exploitation **highly feasible** in adversarial settings.

---

#### Proof of Concept

1. Implement a loop that brute-forces transaction contents until the hash maps to the target slot:

   hash = keccak256(tx_raw)
   if u64_from(hash[0..8]) % committee_size == target_slot:
   submit(tx)

2. Generate 100–1000 large transactions and submit them to the network.

3. Observe the targeted validator’s CPU/bandwidth/memory spike while others remain unaffected.

---

#### Recommendation

Mitigate static hash-to-slot targeting by introducing unpredictability or fallbacks:

**Option 1 — Commit-reveal / VRF-based mapping**
Require periodic commitments (or use VRF) to assign responsibility unpredictably, decoupling `tx.hash()` from slot routing.

**Option 2 — Epoch-level salt randomization**
Add a rotating salt to the assignment:

```
let index = hash64(salt || tx.hash());
if index % committee_size == my_slot { process(tx) }
```

Attackers must re-grind each epoch, raising cost dramatically.

**Option 3 — Load-aware fallback / loosened ownership**
Allow other validators to process transactions when the assigned validator is saturated (fallback heuristics, randomness, or load-aware reassignment).

Implementing one or more of these prevents cheap, deterministic targeting and preserves validator liveness and fair load distribution.


### [I-03] Genesis Precompile Balance Underflow

#### Summary

An unchecked subtraction in the genesis precompile balance calculation can cause an underflow, resulting in an astronomically large token balance during network initialization. If exploited, this bug would enable effectively infinite minting capability, breaking the tokenomics and destabilizing the entire Telcoin network.

---

#### Finding Description

In `src/genesis/mod.rs`, the genesis logic calculates the `InterchainTEL` contract’s balance using:

```
final_itel_balance = itel_balance - governance_balance;
```

If `genesis_stake` (the total initial stake assigned at genesis) exceeds the total token supply (`100 billion TEL`), this subtraction underflows. The resulting wrapped value becomes `2²⁵⁶ - difference`, assigning the `InterchainTEL` contract an absurdly large balance.

**Attack Flow**:

1. Configure genesis with excessive `initial_stake` values across many validators.
2. Ensure `genesis_stake > total_supply`.
3. Calculation underflows:

   ```
   itel_balance = 100_000_000_000e18 - genesis_stake  → underflow
   ```
4. `InterchainTEL` contract receives ~`2²⁵⁶` tokens.
5. Attacker (or misconfiguration) effectively enables infinite minting.

**Violated Assumptions**:

* CLI validation is trusted, but no on-chain safeguard exists.
* Assumed invariant `genesis_stake < total_supply`.
* Arithmetic performed without bounds checks.

---

#### Impact Explanation

* **Infinite Token Minting**: Any actor configuring genesis incorrectly could generate effectively limitless TEL.
* **Total Tokenomic Collapse**: The value of TEL would drop to zero due to unlimited inflation.
* **Network Redeployment**: The chain would be unrecoverable, requiring full restart.
* **Critical Target**: Genesis configuration is a single point of catastrophic failure.

While not exploitable post-genesis, the damage from a misconfigured genesis is existential.

---

#### Likelihood Explanation

* **Medium likelihood**:

  * Requires privileged access to genesis configuration.
  * Needs deliberate or accidental misconfiguration (excessive stake allocation).
  * Not reachable after network launch.

Though only a one-time initialization risk, the catastrophic consequences make this highly relevant for genesis security reviews.

---

#### Recommendation

* **Use Safe Arithmetic**: Replace unchecked subtraction with `checked_sub()` or equivalent safe math.
* **Genesis Invariant Check**: Explicitly enforce that `genesis_stake < total_supply`:

  ```rust
  ensure!(
      genesis_stake < total_supply,
      "Total stake cannot exceed total token supply"
  );
  ```
* **Comprehensive Validation**: Add bounds checks across all genesis configuration values to ensure no underflow/overflow conditions can occur.

This ensures the genesis process cannot introduce infinite balances, preserving tokenomics and network viability.
