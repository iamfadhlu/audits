## AQUARIUS
[Contest Details](https://cantina.xyz/competitions/990ce947-05da-443e-b397-be38a65f0bff)

### [L-01] Gas Griefing Vulnerability Due to Unbounded Token Count in Stable-Swap Pools

#### Summary
The stable-swap pool allows an unbounded number of tokens, enabling gas griefing attacks where excessive tokens inflate gas costs, potentially causing denial-of-service.

#### Finding Description
In `liquidity_pool_router/src/contract.rs`, the `init_stableswap_pool` function permits the creation of a stable-swap pool with a tokens: `Vec<Address>` parameter without enforcing an upper limit on the number of tokens. Operations like `deposit` and `swap` iterate over this token list to perform transfers, reserve updates, and fee calculations (e.g., loops over tokens for each operation). Each iteration involves gas-intensive actions: external calls (`test_token::Client::new().transfer()`), storage updates (`set_pool_reserves in storage.rs`), and arithmetic operations.

An attacker can exploit this by initializing a pool with hundreds of tokens, causing subsequent operations to consume excessive gas. For example, a deposit operation for a pool with 500 tokens would require 500 token transfers and storage updates, potentially exceeding Soroban’s transaction gas limit or making the operation prohibitively expensive. Legitimate users may face transaction failures or unaffordable gas costs, effectively resulting in a denial-of-service (DoS) attack. The vulnerability requires an attacker to create a malicious pool, which may be gated by access controls or fees, but the absence of a token limit makes it feasible if pool creation is permissionless.

#### Impact Explanation
Gas griefing can render the pool unusable by inflating gas costs, preventing users from performing core actions like swaps or deposits. This disrupts the protocol’s functionality, erodes user trust, and may lead to financial losses if users cannot access liquidity. The widespread impact on all pool users justifies a High impact rating.

#### Likelihood Explanation
The permissive design of the `init_stableswap_pool` function increases the likelihood to Medium, as no token limit is enforced.

#### Recommendation
Introduce a maximum token limit in the `init_stableswap_pool` function and add checks in `deposit` and `swap` to ensure the token count remains within safe bounds. A reasonable limit (e.g., 8 tokens) aligning with curve stable-swap best practices (e.g., Curve Finance).