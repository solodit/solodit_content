**Auditors**

Shaflow01

0xluk3

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/029_CODESPECT_TOKENTABLE_SUI.pdf)

# Findings

## Critical Risk

### [C-01] The send_tokens(...) and mark_claimed(...) functions lack access control

**Files:** [`base_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/base_distributor.move#L144)

**Description:**

During the token claim process, the `mark_claimed(...)` function is called to mark tokens as claimed to prevent replay, and then the `send_tokens(...)` function is called to transfer tokens from the `Distributor` treasury to the recipient.

```move
public fun mark_claimed<T>(
    distributor: &mut Distributor<T>,
    claim_id: vector<u8>
) {
    assert!(!is_claimed(distributor, claim_id), E_CLAIM_ALREADY_CLAIMED);
    table::add(&mut distributor.claimed_claims, claim_id, true);
}

public fun send_tokens<T>(
    distributor: &mut Distributor<T>,
    recipient: address,
    amount: u64,
    ctx: &mut TxContext
) {
    let token_balance = sui::balance::split(&mut distributor.token_balance, amount);
    let tokens = sui::coin::from_balance(token_balance, ctx);
    transfer::public_transfer(tokens, recipient);
}
```

However, both functions lack access control, allowing anyone to call them directly.

**Impact:** A malicious actor can call `mark_claimed(...)` before a user claims, marking the tokens as claimed and causing the user’s claim to fail. They can also call `send_tokens(...)` directly to steal all unclaimed tokens from the `Distributor` treasury.

**Recommendation:** Use `public(package)` to restrict these functions so they can only be called by this package.

**Status:** Fixed

**Client response:** [44f9228953fb383f3f356061ba9e245e22389d8d](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/44f9228953fb383f3f356061ba9e245e22389d8d) Note: `public(friend)` is deprecated.

## Medium Risk

### [M-01] The claim(...) and claim_with_fees(\.\.\.\.) functions lack token type checks

**Files:** [`fungible_token_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_distributor.move#L47), [`fungible_token_with_fees_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_with_fees_distributor.move#L57)

**Description:**

In the `claim(...)` function, there is no check to verify whether the fee token type provided by the caller matches the type configured in `fee_collector`, allowing the caller to use a self-created worthless token to pay the fee.

```move
public entry fun claim<T, F>(...) {
    //...
    assert!(option::is_some(&fee_collector_addr), E_FEE_COLLECTOR_NOT_SET);
    let base_fee = fee_collector::get_fee(fee_config, distributor_address);
    let expected_amount = base_fee * multiplier;
    assert!(coin::value(&fee_payment) == expected_amount, E_INCORRECT_FEES);
    fee_collector::collect_fee(fee_config, fee_payment, ctx);
}
```

In the `claim_with_fees(...)` function, although the fee token is arbitrarily specified by the `authorized_signer`, the signed data contains only the fee amount and no fee token type, making it impossible to verify whether the token paid by the user matches the `authorized_signer`’s expectation.

```move
// Claim data structure with fees
public struct ClaimDataWithFees has drop {
    claimable_timestamp: u64,
    claimable_amount: u64,
    fees: u64
}
```

**Impact:** Users can evade paying token claim fees to the project team.

**Recommendation:** It is recommended that the `claim(...)` function check whether the fee token provided by the user matches the type configured in `fee_collector`.

For `claim_with_fees(...)`, it is recommended to include the fee token type in the signature fields so that the `claim_with_fees` function can perform the check.

**Status:** Fixed

**Client response:** [355c59dae35234a78a9f00893bb3ab5814875dcf](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/355c59dae35234a78a9f00893bb3ab5814875dcf)

### [M-02] Distributor has not been assigned a fee collection type

**Files:** [`base_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/base_distributor.move#L17)

**Description:**

The current system has two fee collection methods: one is managed by `OwnerCap` through `fee_collector`, which configures the fee token and fee amount; the other allows the `authorized_signer` in each `Distributor` object to arbitrarily specify it in the signed data.

```move
public entry fun claim<T, F>(...) {
    //...
}

public entry fun claim_with_fees<T, F>(...) {
    //...
}
```

However, the `Distributor` object does not have a field distinguishing the fee collection method. As a result, the `authorized_signer` can choose either method or mix them at will.

**Impact:** This causes fee collection to be outside the control of the project team and allows the token issuer to specify it arbitrarily. The project team may suffer losses from the fees.

**Recommendation:** It is recommended to add a field in the `Distributor` to distinguish the fee collection type.

**Status:** Acknowledged

**Client response:** Since fee value will be baked into the signature for claim type `claim_with_fee`, mixed up usaged will cause claim to fail. So there is no risk of fee loss.

**CODESPECT fix review:** The issue in the report is not that users can freely use signatures from the `authorized_signer`—because mixed-up usage would cause the claim to fail—but rather that the `authorized_signer` (controlled by the token distributor) can arbitrarily assign signature types. For example, the project team may want a certain distributor to use the fees configured in `fee_collector`, but the `authorized_signer` could choose to distribute some signatures related to `ungible_token_with_fees_distributor` with a fee of 0, allowing their users to avoid paying fees to the project team. If the project team is willing to accept this, the issue will be marked as acknowledged.

**Client response:** The claim process will be handled by our front-end which will call the right claim type. We will also communicate with `authorized_signer` (the token distributor) in advance to make sure the correctness of signature.

## Low Risk

### [L-01] The fee collector is optional on Distributor creation, but obligatory on claiming leading to temporary DoS

**Files:** [`fungible_token_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_distributor.move), [`fungible_token_with_fees_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_with_fees_distributor.move), [`base_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/base_distributor.move)

**Description:**

Distributors can be created without a fee collector (its an `Option<address>`), however, the claiming routine enforces the existence of such, otherwise the claims revert. This allows for the creation of default broken Distributors, where claiming will not be possible, requiring Owner intervention to set a fee collector in order to restore the Distributor state.

```move
// In claim routine:
assert!(option::is_some(&fee_collector_addr), E_FEE_COLLECTOR_NOT_SET);
[...]
// In the distributor creation:
public fun set_fee_collector<T>(
    distributor: &mut Distributor<T>,
    fee_collector: Option<address>,
    _owner_cap: &OwnerCap
)
```

**Impact:** It is possible to create non-functional distributors which will not allow claiming, leading to partial DoS condition until manual intervention is performed.

**Recommendation:** Enforce setting a fee collector on Distributor creation or allow Distributors without fee collectors (depending on the business needs of the protocol).

**Status:** Fixed

**Client response:** The configuration will be done manually and carefully managed to avoid human error.

### [L-02] The token distributor cannot control distribution parameters and cannot withdraw undistributed tokens

**Files:** [`base_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/base_distributor.move#L85)

**Description:**

The `set_base_params(...)` function sets token distribution parameters, including start and end times, etc. The `withdraw(...)` function is used to claim the tokens that the issuer deposited into the `Distributor`. Both functions can only be called by the project team (`OwnerCap` holder), so the token issuer cannot call them freely. This is inconsistent with implementations on other chains.

```move
public fun set_base_params<T>(
    distributor: &mut Distributor<T>,
    start_time: u64,
    end_time: u64,
    authorized_signer: address,
    _owner_cap: &OwnerCap
) {
    //...
}

public entry fun withdraw<T>(
    distributor: &mut Distributor<T>,
    _owner_cap: &OwnerCap,
    amount: Option<u64>,
    ctx: &mut TxContext
) {
    //...
}
```

**Impact:** This permission restriction reduces the token issuer’s flexibility in distribution, as they cannot freely start or end a distribution nor claim back undistributed tokens on their own.

**Recommendation:** It is recommended to create a permission credential object for each `Distributor`. This object should include a field pointing to the `Distributor` ID to distinguish credentials for different distributors.

**Status:** Fixed

**Client response:** [Github Commit](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/b5205b92c8321be018f2ca3e381f9ac89822fc8f)

**CODESPECT fix review:** It is recommended to set the permission for `set_fee_collector` to `AdminCap`. Although this field currently has no effect, in other versions the `fee_collector` is set by the administrator.

We also noticed that in the new version of the code, a `project_id` is assigned to the distributor. Currently, creating a distributor requires no permission. If a malicious actor were to preemptively occupy a `project_id`, would there be any impact, considering that the permissions of the two functions have now been moved to the creator?

If the `project_id` has a special meaning or purpose and is not randomly generated, and if it is not associated on the backend after the transaction, then it is recommended that the `AdminCap` first assigns the associated `project_id` → creator, and then the creator can create the distributor and freely configure its parameters.

**Client response:**

1. We made another update that eliminates the need to set `fee_collector`. Now all contract calls will all refer to the global `FeeCollectorConfig`. This part is handled by our front-end;
2. `projec_id` has no special meaning so it is fine;

Commit: [b40f93884b1c21d4bcf780f4d06c5964a3b4eca6](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/2ffa820d1809cc3b2d5e3b3cb87cef54537b50f1)

We made another update that requires each distribution to pass in a fee collector reference during creation. Also changed `get_distribution_info` to public entry to facilitate front-end. [3f3f05504e458fee6ce0163673fec0582ca7c3af](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/3f3f05504e458fee6ce0163673fec0582ca7c3af)

## Informational

### [I-01] Lack of one-time witness may allow objects created in the init(...) function to be created after upgrade

**Files:** [`ownable.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/ownable.move#L9), [`fee_collector.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fee_collector.move#L44)

**Description:**

For objects that need to ensure global uniqueness, we typically create them in the `init` function because the `init` function is only executed once during package deployment. For example, the `FeeCollectorConfig` object in `fee_collector` and the `OwnerCap` object in `ownable`.

However, since a one-time witness is not used, there is no guarantee that these objects are globally unique. Subsequent upgrades could introduce new functions to create these objects.

**Impact:** `OwnerCap` and `FeeCollectorConfig` may no longer remain globally unique after an upgrade, introducing potential risk such as permission escalation and configuration conflicts.

**Recommendation:** It is recommended to use a one-time witness for objects that need to ensure global uniqueness.

**Status:** Fixed

**Client response:** [Github Commit](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/79fe293de702b9c0542800ad00860b5abd839e5e)

**CODESPECT fix review:** Not fixed. It seems that the `witness` is not actually being used. To ensure the uniqueness of the object, the `witness` should be used in a manner like this.

```move
public struct OWNABLE has drop {}

public struct OwnerCap<phantom T> has key {
    id: UID,
}

fun init(witness: OWNABLE, ctx: &mut TxContext) {
    // The witness proves this is the first and only time this is called
    assert!(sui::types::is_one_time_witness(&witness), 0);

    transfer::transfer(OwnerCap<OWNABLE> {
        id: object::new(ctx),
    }, tx_context::sender(ctx));
}
```

**Client response:** [163eb236b6725a04efb29316c1fe1e7640b7ae31](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/163eb236b6725a04efb29316c1fe1e7640b7ae31)

### [I-02] version string could be a single named constant instead of 2 different strings

**Files:** [`fee_collector.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fee_collector.move#L58), [`ownable.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/ownable.move#L14)

**Description:**

In both the `fee_collector` and `ownable` modules, the version function uses the same version string (converted via `string::utf8`).

```move
module ecdsa_token_distributor_sui::ownable {
    //...
    public fun version(): String {
        string::utf8(b"0.1.0")
    }
    //...
}

module ecdsa_token_distributor_sui::base_distributor {
    //...
    public fun version(): String {
        string::utf8(b"0.1.0")
    }
    //...
}
```

**Impact:** Defining the same version string in two separate places creates maintenance overhead, as upgrading the version requires modifying two hardcoded instances in the code.

**Recommendation:** It is recommended to optimise this by using a single named constant instead of redundantly defining two separate strings.

**Status:** Fixed

**Client response:** [2f6b14dc43be6cfd664eaa0ff6b884393d1f0898](https://github.com/EthSign/ecdsa-token-distributor-sui/commits/2f6b14dc43be6cfd664eaa0ff6b884393d1f0898)

### [I-03] Redundant get_distributor_info function calls

**Original severity:** Best Practices

**Files:** [`fungible_token_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_distributor.move#L82), [`fungible_token_distributor.move`](https://github.com/EthSign/ecdsa-token-distributor-sui/tree/5536d80395d269b7d3392b20a924cbcae7a86344/sources/fungible_token_with_fees_distributor.move#L61)

**Description:**

In the `claim(...)` and `claim_with_fees(...)` functions, `get_distributor_info(...)` is called multiple times to retrieve the same data, and the number of redundant calls increases with the number of loop iterations.

```move
public entry fun claim<T, F>(...) {
    let (_, _, _, _, _, paused, _, _, _) = base_distributor::get_distributor_info(distributor);
    //...
    while (i < len) {
        //...
        let (_, authorized_signer, _, _, _, _, _, _, _) = base_distributor::get_distributor_info(distributor);
        //...
    };
    let multiplier = if (delegate_mode) len else 1;
    let (distributor_address, _, _, _, _, _, fee_collector_addr, _, _) =
        base_distributor::get_distributor_info(distributor);
    //...
}

public entry fun claim_with_fees<T, F>(...) {
    let (_, _, _, _, _, paused, _, _, _) = base_distributor::get_distributor_info(distributor);
    //...
    while (i < len) {
        //...
        let (_, authorized_signer, _, _, _, _, _, _, _) = base_distributor::get_distributor_info(distributor);
        //...
    };
    let (_, _, _, _, _, _, fee_collector_addr, _, _) = base_distributor::get_distributor_info(distributor);
    //...
}
```

**Impact:** Such redundant calls increase gas consumption and reduce code readability.

**Recommendation:** It is recommended to retrieve all required variables in a single function call and use them directly in subsequent operations.

**Status:** Fixed

**Client response:** [2f6b14dc43be6cfd664eaa0ff6b884393d1f0898](https://github.com/EthSign/ecdsa-token-distributor-sui/commit/2f6b14dc43be6cfd664eaa0ff6b884393d1f0898)
