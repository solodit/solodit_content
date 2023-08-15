**Auditors**

[AuditOne](https://twitter.com/auditone_team)

# Findings

## High Risk

### Block Reorg Can Allow For Double Spending

**Description**: 

Block reorg, also known as blockchain reorganization, is a situation where a competing chain replaces the main blockchain. This can happen when multiple miners find valid blocks at the same time, and the network has to decide which block to include in the blockchain. In some cases, the network may choose to include a block that is not in the main blockchain, resulting in a reorganization of the chain.

**Recommendations:**

To mitigate the risk of block reorgs, the Fast Bridge project may need to implement additional measures, such as waiting for multiple confirmations before proceeding with token transfers or implementing a fallback mechanism in case of a block reorg.



### Potential for Race Condition between Unlock Time and Proof Verification leading to Double Spending

**Description:** 

In the Fast Bridge project, a race condition between unlock time and proof verification can lead to double spending. Here's how it can happen:

When a user sends tokens from Ethereum to NEAR, the tokens are locked on the Ethereum side, and the LP-Relayer is responsible for releasing the tokens on the NEAR side once the proof of transaction has been received. The LP-Relayer has a specific time window in which to release the tokens, which is set by the sender when they initiate the transaction.

**Recommendations:**

To prevent this from happening, it's important that the LP- Relayer is trusted and secure, and that there are appropriate measures in place to verify that the proof of transaction is valid before the tokens are released. This can include using secure verification processes, and having multiple parties involved in verifying the proof before the tokens are released.



### User can drain all funds by calling withdraw multiple times Severity: High

**Description:**

Currently withdraw function transfer tokens and after its success, the callback function decreases user's balance. As main call and callback handler are independent transaction, a malicious user can call withdraw repeatedly, before callback function called.

As user's balance is still not decreased, user can still transfer tokens.

**Recommendations:**

We should follow CEI pattern in this case.

We should move decrease\_balance to withdraw, above ft\_transfer.



### Malicious user can double-unlock his locked funds Severity: High

**Description:**

Currently unlock function checks whether pending transfer of the nonce does exist or not, after its success, the callback function increases balance and remove transfer.

In Near, main call and callback handler are independent transaction, so a malicious user can call unlock repeatedly, before callback function finished.

As pending transfer is not removed yet, user can still pass the check in second call.

Second call of remove\_transfer in callback will not panic but only return None.

As a result, user can double-unlock his funds.

**Recommendations:**

We should check if the nonce still exists in the pending\_transfers at unlock\_callback.



## Medium Risk

### Subscribed message may get lost Severity: Medium

**Description:**

The subscribe function will stop if there is any error while sending the pubsub\_msg.


Restarting the subscribe function may take time

Within that time all published events would get lost and wont be processed.

**Recommendations:**

Probably an offchain component can keep track of all lost event messages and Admin could send those lost event messages so that they could be processed

Status: Resolved



### Redis db connection issue - Relayer fund loss Severity: Medium

**Description:**

User A use fast bridge to get token on ETH

Relayer executes the transaction on ETH but due to redis issue, the PENDING\_TRANSACTIONS entry could not be made in Redis

This causes Relayer to be unaware about this issue and Relayer now wont issue lp\_unlock on near side

After bridge request expire user can unlock the token. This means user gets both token on eth and near side

**Recommendations:**

 Revert if redis connection issue is present Status: Resolved.



### Lack of Validation for valid\_till\_block\_height on FastBridge Service

**Description:** 

The FastBridge Service, which is responsible for managing transfers between the NEAR and Ethereum networks, does not validate the valid\_till\_block\_height parameter. This parameter is used to set an expiration block height for the transfer, and if not validated properly, it could result in transfers being processed after they have expired.

**Recommendations:**

To address this issue, it is recommended that the ![ref4]FastBridge Service implement proper validation for the valid\_till\_block\_height parameter. This could include checks to ensure that the current block height is not greater than the valid\_till\_block\_height value.



## Low Risk

### Missing comparison between lock\_time\_min and lock\_time\_max in set\_lock\_time function

**Description:**

In the set\_lock\_time function, the lock\_time\_min and lock\_time\_max values are not compared before setting the lock\_duration. This might lead to a situation where lock\_time\_min is greater than lock\_time\_max, which could be an invalid state for the intended logic of the application.

**Recommendation:** 

To prevent potential issues with invalid lock duration configurations, we recommend adding a comparison between lock\_time\_min and lock\_time\_max before updating the lock\_duration. If lock\_time\_min is greater than lock\_time\_max, the function should return an error or panic to indicate that the provided values are invalid.



### Lack of Check for Same token\_eth and recipient in the NEAR Contract

**Description:** 
In the provided code snippet, a JSON object is created, containing the details for a transfer, including the token\_eth and recipient fields. However, there is no check to ensure that the token\_eth and recipient fields are not the same. Allowing the same value for both fields could lead to potential issues in the contract's execution, as it might not be the intended behavior for a valid transfer.

**Recommendations:**

To address this issue, it is recommended that the NEAR contract includes a check to ensure that the token_eth and recipient fields are not the same.



## Informational

### Missing comments on Code 

**Description:**

Most of the contract functions are missing comments.

**Recommendations:**

 Recommended to add comments on all contract functions.



### Unexpected token unlock could happen Severity: Quality Assurance

**Description:**

As per docs, max of valid\_till or valid\_till\_block\_height is always taken to derive the unlock time: "valid\_till\_block\_height: Option — the same as valid\_till, but in block height, not in nanoseconds. If both values are provided, tokens will be locked on the max of the two values. (In that stage for User only None value makes sense)".

But seems like valid\_till\_block\_height is not actually used while checking valid time validity

As we can see in below function unlock time validation is only performed on valid\_till parameter and valid\_till\_block\_height param is not used.

**Recommendations:**

valid\_till\_block\_height should also be used to ![ref5]derive the unlock time.



### Improve handling of CheckToken case in check\_whitelist\_token\_and\_account function

**Description:**

In the check\_whitelist\_token\_and\_account function, the CheckToken variant of the WhitelistMode enum has an empty block, which might be unclear to readers of the code. This variant is intended to indicate that only the token needs to be checked against the whitelist, and no action is required for the associated account. However, the current implementation with an empty block might not effectively communicate this intent.

**Recommendations:**

 To improve the clarity and maintainability of the code, we recommend using the => {} pattern in the match statement for the CheckToken variant. This pattern makes it explicit that no action is needed for this case. Alternatively, you can add a comment within the empty block to explain why no action is required. Here's the updated match statement:


### Unnecessary Initialization verification Severity: Quality Assurance

**Description:**
The require! statement in the new function is used to check if the contract has already been initialized before. However, the function also uses the init macro which is responsible for initializing the contract. Since the init macro can only be called once per contract, there is no need for an additional check using require! to verify whether the contract has been initialized before.

**Recommendations:**

It is recommended to remove the unnecessary require! statement to ensure a cleaner and more efficient codebase.



### Lack of Configurability for Timeouts in Fast Bridge Service Severity: Quality Assurance
**Description:**

The Fast Bridge project has a lack of configurability for timeouts related to transactions. In the code snippet provided, there is a hardcoded timeout of 60 seconds, which may not be sufficient for all transactions. This lack of configurability could result in delays or even failures of transactions, especially in cases where longer confirmation times are required.

**Recommendations:**

To address this issue, it is recommended ![ref5]that the Fast Bridge project implement a more configurable approach for timeouts related to transactions.

Status: Resolved.



### Wrong parameter on lp\_unlock near call at unlock\_tokens in Relayer

**Description:**

From Relayer, it passes { nonce, proof } as param to lp\_unlock call.

However, actual near implementation has just one param proof.![ref5]

**Recommendations:**

As proof includes nonce and everything looks good at Near side, we should keep consistency and remove nonce from lp\_unlock call from Relayer.



### Limitations and Risks for Users in the Fast Bridge Project Severity: Quality Assurance

**Description:** 

The Fast Bridge project allows for the transfer of tokens between the NEAR Protocol and Ethereum networks. However, there are several limitations and risks for users. For example, if there is no relayer available to handle the transaction, the tokens will be locked until the valid\_till time, and users will need to contact support to unlock stuck tokens on NEAR. The transaction size is limited by relayer liquidity, and only tokens from the whitelist can be transferred. Additionally, the price is significantly higher than the original bridge.

On the other hand, The Fast Bridge project relies on LP-Relayers to hold locked tokens on the Ethereum side and release them on the NEAR side. However, there are several risks and limitations for relayers. For example, there is a risk of double unlock if the relayer already transferred tokens on the Ethereum side and switches off for a while. The relayer is an off-chain part that needs extra maintenance, including increasing liquidity and key management protection. The relayer can also have issues with the internet or server uptime.

**Recommendations:**

To mitigate these risks, it is recommended ![ref5]that the Fast Bridge project implement a more decentralized approach that reduces reliance on the relayer and provides greater flexibility and scalability for users.



### Out-of-date Rust Crate Detected with Cargo Audit Severity: Quality Assurance

**Description:** 

A recent cargo audit has detected an out-of-date Rust crate within the project. Cargo audit is a tool that analyzes the Rust project's dependency tree and reports any known security vulnerabilities or outdated dependencies. Using outdated crates can introduce potential risks and negatively affect the performance, stability, and security of the application.

**Recommendations:** 

To address this issue, it is recommended to update the out-of-date crate to its latest version, ensuring that any security vulnerabilities, bug fixes, or new features are incorporated into the project. 

