
### [L-01] Program Cache Poisoning via Malicious Account Data (#220)

#### Summary
The SVM engine is vulnerable to a Denial of Service (DoS) attack due to improper deserialization of program account data, leading to crashes.

---

#### Finding Description
The functions `load_bpf_loader_upgradeable_program_to_cache` and `load_bpf_loader_program_to_cache` within `engine/src/processor.rs` are susceptible to **program cache poisoning**.  

The vulnerability arises from the use of:  

```rust
bincode::deserialize(program_account.data()).unwrap()
```

without adequate error handling or size checks prior to deserialization.

If `program_account.data()` contains malformed or corrupted data, the `unwrap()` call causes the SVM engine to panic and crash.

**Attack Scenario:**

1. **Crafted Account**: An attacker creates an account with `bpf_loader_upgradeable` as its owner and populates its data with malformed input (e.g., invalid length, corrupted serialization).
2. **Trigger Loading**: The attacker ensures the malicious account’s `program_id` is passed to `fill_cache` or `load_program_to_cache`, either via system configuration or by submitting a transaction that references it.
3. **Denial of Service**: When the SVM engine attempts to load the malformed program into its internal cache, the `unwrap()` triggers a panic, immediately crashing the instance.

This flaw stems from the unsafe assumption that account data will always be valid and that deserialization failures need not be handled defensively.

---

#### Impact Explanation

The primary impact is **Denial of Service (DoS)**.

* A successful attack crashes the SVM engine instance, halting transaction processing and making it unavailable.
* Repeated exploitation could cause persistent unavailability, disrupting all dependent services.
* While no direct data corruption occurs, prolonged downtime undermines data integrity by preventing consistent transaction ordering or completion.
* Continuous malformed program submissions could also exhaust system resources.

Thus, while classified here as “Low” severity in tracking, the ability for any external actor to crash the entire engine remains a **serious resilience concern**.

---

#### Likelihood Explanation

The likelihood of exploitation is **high** because:

* **Direct Control over Input**: Attackers control the data in program accounts.
* **Lack of Validation**: Use of `.unwrap()` makes the code brittle against malformed input.
* **Common Attack Vector**: DoS via deserialization flaws is a well-known and simple approach.
* **Codebase Precedent**: Other areas of the codebase use safer error handling (e.g., nonce handling), showing this risk could have been avoided.
* **Ease of Triggering**: Loading a program into cache is a standard operation, making the path easily reachable.

---

#### Proof of Concept

*Not required for this contest.*

---

#### Recommendation

Replace the unsafe `unwrap()` call with **robust error handling** that:

* Gracefully handles deserialization failures.
* Logs errors without panicking.
* Prevents malformed data from reaching critical execution paths.

This ensures malformed accounts cannot crash the engine and maintains system availability against adversarial input.



### [L-02] Economic Manipulation via MAX_PROCESSING_AGE Bypass (#221)

#### Summary
The SVM engine contains a critical vulnerability that allows attackers to bypass transaction age limits, enabling the submission of stale transactions that should have been rejected. This flaw can lead to economic manipulation and state inconsistencies.

---

#### Finding Description
The vulnerability resides in the `check_age` function located in `engine/src/verification.rs`, which is described as a *"completely stripped version"* of the intended implementation.  

This function currently returns a hardcoded success response:  

```rust
Ok(CheckedTransactionDetails { nonce: None, lamports_per_signature: 1_000 })
```

without performing any actual checks against the `max_age` parameter or the transaction's age. As a result, the SVM engine does not enforce any limits on transaction age, allowing stale transactions to be processed as if they were fresh.

**Security Guarantees Broken:**

* **Transaction Freshness**: Transactions with outdated `recent_blockhash` values are incorrectly accepted.
* **Replay Attack Prevention**: Stale transactions can be resubmitted for replay attacks, enabling state manipulation or market exploitation.
* **Economic Integrity**: Without age checks, attackers can exploit arbitrage or front-run based on outdated conditions.

**Attack Vector:**

1. **Stale Transaction Creation**: Attacker creates a transaction with an old `recent_blockhash`.
2. **Delayed Execution**: Attacker holds the transaction until favorable conditions emerge.
3. **Exploitation**: Attacker submits the transaction, which is processed successfully despite its staleness.

This vulnerability stems from the placeholder implementation of `check_age`, which was assumed to be replaced with robust validation in the future but currently leaves the system exposed.

---

#### Impact Explanation

* **Economic Manipulation**: Attackers can exploit stale transactions for unfair economic gain.
* **State Inconsistency**: Actions executed on outdated information may conflict with the current system state.
* **Trust Erosion**: Users may lose confidence in the system’s fairness if stale transactions are processed.

Given these consequences, the vulnerability violates fundamental blockchain security guarantees and poses a **significant economic risk**.

---

#### Likelihood Explanation

* **Direct Control Over Input**: Attackers freely craft transactions with old `recent_blockhash` values.
* **Validation Stage Exposure**: The vulnerable `check_age` function is called during transaction validation.
* **Common Attack Vector**: Stale transaction submission is a well-known blockchain attack vector.

These factors make exploitation **highly feasible**.

---

#### Proof of Concept

**Steps for Exploitation:**

1. Generate a transaction with an old `recent_blockhash`.
2. Submit the transaction to the SVM engine.
3. Observe that it is accepted and processed, despite exceeding normal age limits.
4. Use the stale execution to manipulate state or exploit outdated economic conditions.

---

#### Recommendation

Properly implement the `check_age` function to enforce transaction age limits using the `max_age` parameter. Stale transactions must be rejected to maintain integrity, prevent replay attacks, and preserve trust in the system.
