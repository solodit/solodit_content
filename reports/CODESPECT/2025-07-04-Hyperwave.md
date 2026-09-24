**Auditors**

[Talfao](https://x.com/talfao1)

[JecikPo](https://x.com/jecikpo)

[0xMrjory](https://x.com/0xMrjory)

[Shaflow01](https://x.com/shaflow01)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/024_CODESPECT_HYPERWAVE_SOLVER_OFF_CHAIN_BOT.pdf)

# Findings

## High Risk

### [H-01] Insufficient result validation in HyperLiqud SDK response

**Files:** [`hypercore.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/cea4d156cafd2d9a5757d21d85aaea1db81479d6/app/shared/hyperliquid/hypercore.py#L155)

**Description:**

The program uses the Hyperliquid Python SDK to request on-chain data, but the returned data is not properly validated. In the Hyperliquid Python SDK, the functions ultimately construct corresponding POST requests to the Hyperliquid API and then return the response data.

Below snippet from [Hyperliquid API](https://github.com/hyperliquid-dex/hyperliquid-python-sdk/blob/d09e5382bba4d0d105c7617c7ea503a63a7f4f3c/hyperliquid/api.py#L19):

```python
def post(self, url_path: str, payload: Any = None) -> Any:
    payload = payload or {}
    url = self.base_url + url_path
    response = self.session.post(url, json=payload)
    self._handle_exception(response)
    try:
        return response.json()
    except ValueError:
        return {"error": f"Could not parse JSON: {response.text}"}

def _handle_exception(self, response):
    status_code = response.status_code
    if status_code < 400:
        return
    if 400 <= status_code < 500:
        try:
            err = json.loads(response.text)
        except JSONDecodeError:
            raise ClientError(status_code, None, response.text, None, response.headers)
        if err is None:
            raise ClientError(status_code, None, response.text, None, response.headers)
        error_data = err.get("data")
        raise ClientError(status_code, err["code"], err["msg"], response.headers, error_data)
    raise ServerError(status_code, response.text)
```

After receiving the response, the `_handle_exception` function is called to handle the status code. If the status code is less than 400 and the response fails to parse as JSON, it will return an error message in JSON format.

In this case, since HyperCore does not filter or validate this result, it may mistakenly assume that the returned data is valid and proceed with parsing and execution.

**Impact:** If data retrieval fails, the program may continue executing, which can result in incorrect or failed exchange rate updates. For example, when calling `get_spot_balances`, if `spot_user_state` returns an error JSON response, the function will not treat it as a failure. Instead, it will return an empty list, and subsequent operations will continue executing. This may lead to incorrect exchange rate calculations.

**Recommendation:** It is recommended to first check if the error field exists in the response data. If it does, throw an exception to terminate the task.

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22).

## Medium Risk

### [M-01] Incorrect price validity check

**Files:** [`RedstoneStablecoinRateProvider.sol`](https://github.com/SwellNetwork/boring-vault/tree/ce21e49b7a7be06c3a96c3c3ea6982c32e3ff224/src/oracles/RedstoneStablecoinRateProvider.sol)

**Description:**

When checking the price validity of the quote token, the code mistakenly uses `MAX_TIME_FROM_LAST_UPDATE_BASE_FEED` instead of `MAX_TIME_FROM_LAST_UPDATE_QUOTE_FEED`.

```solidity
function getRate() public view returns (uint256 rate) {
    //...
    (, int256 _quoteRate,, uint256 lastUpdatedAtQuote,) = PRICE_FEED_QuoteFeed.latestRoundData();

    if (
        lastUpdatedAtQuote > block.timestamp
            || block.timestamp - lastUpdatedAtQuote > MAX_TIME_FROM_LAST_UPDATE_BASE_FEED
    ) {
        revert MaxTimeFromLastUpdatePassed(block.timestamp, lastUpdatedAtQuote);
    }
    //...
}
```

**Impact:** This causes the quote token price validity period to deviate from the expected duration.

**Recommendation:** Use `MAX_TIME_FROM_LAST_UPDATE_QUOTE_FEED` to validate the freshness of the quote token price.

**Status:** Fixed

**Client response:** Fixed in [PR-6](https://github.com/SwellNetwork/boring-vault/pull/6/files).

### [M-02] Incorrectly assumes that the USDE/USDC rate will not exceed 1 most of the time

**Files:** [`RedstoneStablecoinRateProvider.sol`](https://github.com/SwellNetwork/boring-vault/tree/ce21e49b7a7be06c3a96c3c3ea6982c32e3ff224/src/oracles/RedstoneStablecoinRateProvider.sol)

**Description:**

The `getRate()` function obtains the price from the Redstone price feed and then calculates the rate, but both the price of USDE and the resulting rate are capped at 1.

```solidity
function _calculateRate(uint256 baseRate, uint256 quoteRate) internal view returns (uint256) {
    //...
    uint256 oneQuoteRate = 10 ** quoteRateDecimals;
    uint256 minQuoteRate = quoteRate > oneQuoteRate ? oneQuoteRate : quoteRate;

    uint256 rate = (minQuoteRate * SCALE) / baseRate;

    return rate > ONE ? ONE : rate;
}
```

The team’s response was:

> In some cases, the price of USDE/USDC can be slightly higher than 1. We don’t want to mint more HLP than necessary, especially since we know that prices will revert to 1 at some point.

However, based on actual observations of the Redstone oracle, the price of USDE remains around 1.0000 to 1.0001 for extended periods, while the price of USDC stays around 0.9997 to 1.0000. For example, the USDE price from June 2nd to June 9th consistently stayed above 1. This means that the USDE/USDC ratio should be greater than 1 under stable conditions, not equal to 1 as the team expects. As a result, the exchange rate is underestimated under normal conditions.

**Impact:** Most of the time, the exchange rate will be lower than the actual rate.

**Recommendation:** Adjust the maximum cap slightly higher.

**Status:** Fixed

**Client response:** Fixed in [PR-6](https://github.com/SwellNetwork/boring-vault/pull/6/files) by adding `MAX_RATE`, which can be changed by the governor.

### [M-03] Potential misreporting of vault balance

**Files:** [`hypercore.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/shared/hyperliquid/hypercore.py)

**Description:**

The bot, when calculating the new exchange ratio, takes into account all balances across the ecosystem. One of these balances is the multi-sig balance held in the HLP vault, which is fetched via an API call and returned by the following function:

```python
def get_vault_equities(self, address: str) -> Dict[str, any]:
    request_data = {"type": "userVaultEquities", "user": address}
    equities = self.exchange.info.post("/info", request_data)
    if not equities or len(equities) == 0:
        return {
            "vaultAddress": base_config.hyperliquid.vault_address,
            "equity": "0",
            "lockedUntilTimestamp": 0,
        }
    return equities[0]
```

This function returns only the first element of the equities array. The array itself consists of dictionaries, each representing vault deposits associated with a given wallet.

Although the protocol team has stated that deposits will only be made to the HLP vault, a mistake (e.g., a deposit to a different vault) or an external actor depositing on someone else’s behalf (in the context of Hyperliquid’s closed system) could alter the content of the array. As a result, an incorrect vault balance may be returned, potentially skewing the accounting logic.

*Note: The CODESPECT team cannot confirm whether depositing on someone else’s behalf is possible. Based on the available documentation, this is not allowed or supported, which reduces the severity of the issue.*

**Impact:** Incorrect accounting of vault balances may lead to an inaccurate exchange ratio calculation, ultimately devaluing user shares.

**Recommendation:** Filter and return only the equity associated with the specific HLP vault address to ensure accurate balance reporting.

**Status:** Fixed

**Client response:** Fixed in [0179e8db97501b7b0f29d628ad6df1769432f3cd](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/0179e8db97501b7b0f29d628ad6df1769432f3cd).

## Low Risk

### [L-01] Insufficient offchain filtering for withdrawal requests

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py#L229)

**Description:**

Before processing withdrawal requests, the bot performs offchain filtering to minimize the number of RPC calls. One of these filters involves verifying the validity of a request using the AtomicQueue, specifically via the `isAtomicRequestValid(...)` function.

However, the current implementation of the filtering only checks whether a request has been marked as solved or not. This is insufficient, as it does not reliably indicate whether the request has been *properly* solved.

The filtering logic should be improved by introducing the following checks:

- `offerAmount > 0`: This ensures that the request has actually been filled;
- Deadline not exceeded: Requests whose deadlines have passed should be skipped to avoid unnecessary RPC calls;
- offer token equals the `boringVault` address: If the offer token does not match the expected vault address, the withdrawal will revert. This check is currently missing from the `isAtomicRequestValid(...)` validation;

**Impact:** Failure to implement these checks may lead to unnecessary latency when solving withdrawal requests and could cause transaction reverts.

**Recommendation:** Enhance the offchain filtering logic to include:

- A check for `offerAmount > 0`;
- Validation that the deadline has not been exceeded;
- Confirmation that the offer token matches the `boringVault` address;

**Status:** Fixed

**Client response:** Fixed in [4ca30a19ba0e76c77ceff2c3ce7084133018b189](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/4ca30a19ba0e76c77ceff2c3ce7084133018b189).

### [L-02] Mix of sync/async causes latency in withdrawal processing

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py#L277)

**Description:**

In the `solve_atomic_requests()` function, the following line is used to send a Telegram alert:

```python
asyncio.run(
    self.telegram_bot.send_msg(
        base_config.telegram.group_chat_id,
        f"Boring Vault {self.boring_vault_address} has not enough balance to solve withdrawal requests for {want_contract.symbol()}: "
        f"Required: {want_contract.from_wei(minimum_assets_out)} {want_contract.symbol()} "
        f"Vault balance: {want_contract.from_wei(boring_vault_balance)} {want_contract.symbol()}",
    )
)
```

This implementation mixes asynchronous and synchronous programming, which introduces several concerns:

- `asyncio.run()` creates a new event loop and blocks the current thread until completion.
- It adds unnecessary overhead just for a single async call.
- If the Telegram API call is slow or fails, it can block or slow down the entire withdrawal request handling.

**Impact:** Potential performance degradation or blocking of the withdrawal processing flow due to external API delays.

**Recommendation:**

- Offload the Telegram call using a background thread or queue-based notification mechanism.
- Alternatively, refactor the logic to be fully synchronous with proper try/except handling to isolate potential Telegram errors from the main logic.

**Status:** Fixed

**Client response:** Fixed in [41378fa9b9fbee9b334ee0e27163e060dfa8f0cb](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/41378fa9b9fbee9b334ee0e27163e060dfa8f0cb) by creating new function for sending Telegram message synchronously.

### [L-03] The multi-chain solve_withdrawal logic is too tightly coupled

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py#L541)

**Description:**

Currently, the multi-chain `solve_withdrawal` logic is too tightly coupled. A single-node failure on any one chain can easily cause the entire task to fail, preventing order fulfillment across all chains.

For example:

- `solve_withdrawal` relies on all oracles and rate providers across all chains functioning correctly, and on prices not being outdated. If the oracle for any asset on any chain fails to meet the required conditions, the task will terminate, preventing order execution on all chains. This is because calculating and update the exchange rate involves calling the oracles and rate providers for all assets across all chains. A single oracle failure can prevent order fulfillment across all chains. For instance, if a `rateProvider` on one chain is unable to provide a valid price due to stale price data from its source, orders on other chains will also fail to be fulfilled.
- During the `solve_atomic_requests` process, when fulfilling orders on each chain, if an RPC exception, network instability, or transaction failure occurs while calling a contract on any chain, the process will immediately terminate instead of skipping and continuing to fulfill orders on the remaining chains. This causes partial execution.

```python
def solve_atomic_requests(
    self, chain_id: str, offer: str, want: str, requests: List[AtomicRequest], rate_by_token: Dict[str, int]
):
    // ...
    want_contract.approve_if_needed(solver_account, vault_config.atomic_solver_v3_address, minimum_assets_out)
    self.atomic_solver_v3.redeem_solve(
        chain_id=chain_id,
        atomic_queue_address=vault_config.atomic_queue_address,
        teller_address=vault_config.teller_address,
        offer=offer,
        want=want,
        users=users,
        minimum_assets_out=minimum_assets_out,
        max_assets=max_assets,
    )
    self.transfer_dust_from_solver_to_vault(chain_id)
```

**Impact:** The strong coupling makes the entire system vulnerable to single points of failure.

**Recommendation:** It is recommended to decouple order fulfillment across different chains, allowing partial fulfillment in some cases. Additionally, providing fallback oracles or alternative pricing methods can help reduce the impact when the primary oracle is down.

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22).

## Informational

### [I-01] Additional check for isAtomicRequestValid(...)

**Files:** [`AtomicQueue.sol`](https://github.com/SwellNetwork/boring-vault/blob/cff94e7093a688578d13279748cdaa0d2b308293/src/atomic-queue/AtomicQueue.sol)

**Description:**

The `isAtomicRequestValid(...)` function is used to validate user-submitted withdrawal requests. It performs several checks, such as verifying the user’s balance and ensuring they have approved the Queue contract to spend their offer tokens.

However, the function does not verify whether the offer token is the Boring Vault share token. This check is crucial and should be added to ensure complete validation redundancy, as recommended for offchain filtering.

**Impact:** Invalid requests may delay the processing of valid ones, since this function plays a critical role in the bot’s request validation logic.

**Recommendation:** Add a check to confirm that the offer token is the Boring Vault share token.

**Status:** Fixed

**Client response:** Fixed in [d3d7f57116f1c366062daf6ddd762e18d9df12de](https://github.com/SwellNetwork/hlp-internal-be/commit/d3d7f57116f1c366062daf6ddd762e18d9df12de).

### [I-02] Check user’s offer balance across requests to prevent unnecessary reverts

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py)

**Description:**

Users create atomic requests via `safeUpdateAtomicRequest(...)`, which verifies whether the user has sufficient offer balance and allowance to the `AtomicQueue` contract. This is intended to ensure only valid requests are created and that the user has enough Boring Vault shares.

Later, the Hyperwave solver bot checks request validity again via `isAtomicRequestValid(...)`, which performs similar validations. However, these checks are insufficient and allow for a specific edge case that may cause the bot to raise an error and delay solving of otherwise valid requests. Here’s an example:

- A user holds 10 shares;
- The user submits two requests: A (`want = USDe`) and B (`want = USDT`), both with `offerAmount = 10`;
- Both requests pass the balance and allowance checks at submission time and during bot validation;
- The bot includes both requests in its solving plan;
- The batch for request A is solved successfully;
- When attempting to solve the batch containing request B, the user’s share balance is already depleted (0), causing the transaction to revert;

As a result, the bot encounters a failure when executing the second batch, leading to unnecessary delays in processing valid requests.

**Impact:** Potential failure of the bot and delay in solving requests due to inconsistent request accounting.

**Recommendation:** Consider adding logic on the bot side to track cumulative user balances across all pending requests, and only include those that can be fully covered.

**Status:** Fixed

**Client response:** Fixed in [5eedc610bb1d0e9df64e366067ccbc297cc3f9c0](https://github.com/SwellNetwork/hlp-internal-be/commit/5eedc610bb1d0e9df64e366067ccbc297cc3f9c0).

### [I-03] Improve front-running mitigations for solve(...)

**Files:** [`AtomicQueue.sol`](https://github.com/SwellNetwork/boring-vault/blob/cff94e7093a688578d13279748cdaa0d2b308293/src/atomic-queue/AtomicQueue.sol)

**Description:**

The current design of `AtomicQueue` is prone to front-running attacks, as acknowledged in the `@dev` comment of the `solve(...)` function: _“It is very likely solve TXs will be front run if broadcasted to public mem pools, so solvers should use private mem pools.”_

When `solve` is called, an attacker may update their withdrawal request in the same block to either:

- Gain a more favourable price ratio, or;
- Cause the entire batch to revert by exceeding the `maxAssets` limit supplied by the solver bot;

This introduces a potential attack surface that could disrupt the batch execution or unfairly benefit malicious actors.

To mitigate this, the `AtomicQueue` design should be adjusted to include a delay mechanism. For example, a request must exist in the system for a minimum duration (e.g., X seconds) before it becomes solvable.

**Impact:** Front-running of atomic request updates could lead to unfair price execution for a user or complete transaction failure due to batch reverts.

**Recommendation:** Consider enforce a rule that requests can only be solved after a certain time period has passed since their submission.

**Status:** Fixed

**Client response:** Fixed in [1503d53e5517df71602468a30984860ddd3064e4](https://github.com/SwellNetwork/boring-vault/commit/1503d53e5517df71602468a30984860ddd3064e4) (contract side) and [5eedc610bb1d0e9df64e366067ccbc297cc3f9c0](https://github.com/SwellNetwork/hlp-internal-be/commit/5eedc610bb1d0e9df64e366067ccbc297cc3f9c0#diff-db2ad265c7a4170665800a49c0939fc27aebc72ef3fdf771ac0f8e8321f5fb1aR301) (off-chain bot side).

### [I-04] Missing Database Error Handling

**Files:** [`atomic_request_repo.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/repo/atomic_request_repo.py)

**Description:**

The `AtomicRequestRepo` class lacks error handling for database operations. All database calls (`get_by_id(...)`, `get_by_is_solved(...)`, `create(...)`) can throw SQL exceptions that will propagate unhandled to the calling code.

**Impact:**

- Silent failures — `get_by_id(...)` returns `None` for both “not found” and “database error” scenarios;
- Poor debugging — no logging of database errors;

**Recommendation:** Add comprehensive error handling with proper exception categorization:

```python
def create(self, request: AtomicRequest) -> AtomicRequest:
    try:
        with Session(self.db_engine) as session:
            session.add(request)
            session.commit()
            session.refresh(request)
            return request

    except IntegrityError as e:
        logger.error(f"Constraint violation creating request: {e}")
        if "duplicate key" in str(e).lower():
            raise ValueError("Request already exists")
        raise ValueError("Request violates business rules")

    except OperationalError as e:
        logger.error(f"Database connection error: {e}")
        raise RuntimeError("Database temporarily unavailable")

    except Exception as e:
        logger.error(f"Unexpected database error: {e}")
        raise
```

> Note: This code is provided as a reference example for the types of errors that should be handled.

**Status:** Acknowledged

**Client response:** Acknowledge. Silent failures will be handled in domain logic (e.g. `BoringVaultService` in `boring_vault.py`). For logging, there’s a global exception handler (FastAPI/celery built-in) which will catch and log the error, so we want to keep current version to make the code of `AtomicRequestRepo` short.

### [I-05] Missing alert on unfavourable exchange rate update

**Files:** [`accountant.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/contract/accountant.py#L37)

**Description:**

The `boring_vault` bot updates the exchange rate in the Accountant contract. The contract checks if the new rate is within the allowed bounds. If not, it pauses the vault:

```solidity
function updateExchangeRate(uint96 newExchangeRate) external requiresAuth {
    // ...
    uint64 currentTime = uint64(block.timestamp);
    uint256 currentExchangeRate = state.exchangeRate;
    uint256 currentTotalShares = vault.totalSupply();
    if (
        currentTime < state.lastUpdateTimestamp + state.minimumUpdateDelayInSeconds
            || newExchangeRate > currentExchangeRate.mulDivDown(state.allowedExchangeRateChangeUpper, 1e4)
            || newExchangeRate < currentExchangeRate.mulDivDown(state.allowedExchangeRateChangeLower, 1e4)
    ) {
        state.isPaused = true;
        // ...
```

As the rate is updated multiple times daily, an out-of-bound value may pause the vault unnecessarily. The bot should notify the protocol team (e.g., via Telegram) before applying such a value, so it can be reviewed and, if needed, updated manually.

**Impact:** An invalid exchange rate can pause the vault, potentially causing unexpected liquidations in integrated protocols that use the vault’s liquid HLP.

**Recommendation:** Notify the protocol team of unfavourable updates and skip the update until reviewed. However, the protocol team should ensure the rate stays fresh.

**Status:** Fixed

**Client response:** Fixed in [7ff95c02b0cb1b147bb1f8a4ae89314e001d982a](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/7ff95c02b0cb1b147bb1f8a4ae89314e001d982a) and [5328945b3866847bfed2b224e5c7d6086a017143](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/5328945b3866847bfed2b224e5c7d6086a017143).

### [I-06] Non-unique addresses may cause balance miscalculation

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py)

**Description:**

During the calculation of token balances, the following loops are executed:

```python
# ...
for token_address in base_config.boring_vault.all_token_addresses:
    token_contract = self.token_contract_by_address[token_address]
    balance = token_contract.balance_of(self.boring_vault_address)
    token_balances[token_address] = token_balances.get(token_address, 0) + balance

# Multisig balances on HyperCore
for ms in base_config.boring_vault.multisig_addresses:
    base_token_decimals = self.base_token_contract.decimals()
    # ...
```

The token and multisig addresses are defined in `all_token_addresses` and `multisig_addresses`, respectively—both populated via a configuration file maintained by the protocol team.

Currently, these structures are lists (array type), which may allow accidental duplication of entries. Since each token or multisig address should be processed only once, it would be a best practice to use a set instead. This ensures uniqueness and prevents redundant computation.

**Impact:** If any address is duplicated in the configuration, it may lead to incorrect accounting due to repeated balance aggregation.

**Recommendation:** Change the data structures `all_token_addresses` and `multisig_addresses` from lists to sets to enforce uniqueness automatically.

**Status:** Fixed

**Client response:** Fixed in [d8da0f599a898a4484e22dbb3652f496f671580b](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/d8da0f599a898a4484e22dbb3652f496f671580b).

### [I-07] Note Error

**Files:** [`RedstoneStablecoinRateProvider.sol`](https://github.com/SwellNetwork/boring-vault/tree/ce21e49b7a7be06c3a96c3c3ea6982c32e3ff224/src/oracles/RedstoneStablecoinRateProvider.sol)

**Description:**

During the assignment in the constructor function, there is an error in the note. The note states that the Default Lower Bound is 5 bps, but it should actually be 9995 bps.

```solidity
constructor(...) {
    //...
    // Default Lower Bound is 5 bps
    lowerBound = 10 ** RATE_DECIMALS * 9995 / 10_000;
    //...
}
```

**Impact:** Incorrect comments may mislead developers and cause difficulties in code maintenance.

**Recommendation:** Change 5 bps to 9995 bps.

**Status:** Fixed

**Client response:** Fixed in [PR-6](https://github.com/SwellNetwork/boring-vault/pull/6/files).

### [I-08] Exponential calculations may suffer from precision loss

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/cea4d156cafd2d9a5757d21d85aaea1db81479d6/app/domain/boring_vault/service/boring_vault.py#L208)

**Description:**

When performing exponential calculations, if `token_contract` decimals is less than `base_token_contract` decimals, the exponent will become negative. Exponentiating 10 with a negative number will result in a value less than 1, and direct computation may lead to precision loss.

```python
def calc_withdrawable_rate_by_token(self, chain_id: str, token_address: str, exchange_rate_lower_bound: int) -> int:
    // ...
    return int(
        Decimal(exchange_rate_lower_bound)
        * Decimal(10 ** (token_contract.decimals() - base_token_contract.decimals()))
        / rate_vs_base
    )

def normalize_balances_by_oracle(self, chain_id: str, token_balances: Dict[str, Decimal]) -> Decimal:
    // ...
    total_assets += (
        balance * rate_vs_base * Decimal(10 ** (base_token_contract.decimals() - token_contract.decimals()))
    )

    return total_assets

def normalize_balances_by_rate_provider(self, chain_id: str, token_balances: Dict[str, Decimal]) -> Decimal:
    // ...
    total_assets += balance * Decimal(rate) / Decimal(10 ** (2 * decimals - base_token_contract.decimals()))

    return total_assets

def get_oracle_rate_vs_base_upper_bound(self, chain_id: str, token_address: str) -> Decimal:
    // ...
    rate_vs_base = max(
        Decimal(oracle_rate)
        / Decimal(base_token_oracle_rate)
        * Decimal(10 ** (base_token_oracle.decimals() - oracle.decimals())),
        Decimal(1),
    )
    logger.info(f"Oracle rate upper bound vs base for {token_contract.symbol()}: {rate_vs_base}")

    return rate_vs_base
```

**Impact:** Performing operations directly between a negative number and 10 may lead to some precision loss.

**Recommendation:** It is recommended to convert to `Decimal` before performing exponential calculations. For example, change the following statement:

```python
int(
    Decimal(exchange_rate_lower_bound)
    * Decimal(10 ** (token_contract.decimals() - base_token_contract.decimals()))
    / rate_vs_base
)
```

To the following:

```python
int(
    Decimal(exchange_rate_lower_bound)
    * (Decimal(10) ** Decimal(token_contract.decimals() - base_token_contract.decimals()))
    / rate_vs_base
)
```

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22).

### [I-09] Some calls within the WithdrawalSolver task lack a retry mechanism

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/cea4d156cafd2d9a5757d21d85aaea1db81479d6/app/domain/boring_vault/service/boring_vault.py#L576)

**Description:**

The WithdrawalSolver task involves many operations that rely on on-chain data retrieval and calls via underlying RPC, as well as database connections. If these operations temporarily fail due to transient network instability or other issues, the program often throws exceptions directly without an appropriate retry mechanism with backoff delay. This will cause the WithdrawalSolver task to terminate immediately.

For example, in the `approve_if_needed` function, it attempts to send the transaction only once, and if it fails, it directly raises an exception without retrying.

```python
def approve_if_needed(self, owner: LocalAccount, spender: str, amount: int) -> Optional[str]:
    // ...
    signed_tx = self.w3.eth.account.sign_transaction(tx, owner.key)
    tx_hash = self.w3.eth.send_raw_transaction(signed_tx.raw_transaction)
    tx_hash_str = add_0x_prefix(tx_hash.hex())
    logger.info(f"Transaction sent: {tx_hash_str}")

    # Wait for transaction receipt (success)
    receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
    logger.debug(f"Transaction mined: {receipt}")
    ...
```

**Impact:** The WithdrawalSolver task has a relatively long scheduling interval—approximately one hour. As a result, if the task execution fails due to transient instability, it may lead to withdrawal delays and increase the processing load during the next execution.

**Recommendation:** It is recommended to implement an appropriate delayed retry mechanism for failures in database and on-chain RPC calls.

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22) by adding auto retry to Celery tasks.

### [I-10] update_exchange_rate_all_chains may result in partial exchange rate updates

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/cea4d156cafd2d9a5757d21d85aaea1db81479d6/app/domain/boring_vault/service/boring_vault.py#L559)

**Description:**

The `ExchangeRateTask`, which is responsible for updating the on-chain `exchange_rate`, calls the `update_exchange_rate_all_chains` function. This function attempts to update the exchange rate for each chain.

However, if any one of the chains encounters an issue during the process, it will cause the program to throw an exception and stop execution, resulting in only partial exchange rate updates.

For example, if the exchange rates for two chains have already been updated but the third chain’s contract is paused and throws an exception, the task will immediately stop. The exchange rates for the first two chains have been updated, but the remaining chains will not be updated.

```python
def update_exchange_rate_all_chains(self):
    """Update the exchange rate of the vault across all chains."""
    for chain_id, new_rate in self.get_exchange_rate_upper_bound_by_chain().items():
        self.accountant.update_exchange_rate(chain_id, new_rate)

def update_exchange_rate(self, chain_id: str, new_rate: int) -> str:
    # Check current state
    current_state = self.get_accountant_state(chain_id)
    if current_state.is_paused:
        self.telegram_bot.try_send_msg(
            chat_id=base_config.telegram.group_chat_id,
            text=f"Exchange rate update failed on {chain_id}: Accountant is paused.",
        )
        raise ExchangeRateError(f"Accountant is paused on {chain_id}")
    if current_state.last_update_timestamp + current_state.minimum_update_delay_in_seconds > now_seconds():
        logger.info(f"Accountant last update timestamp is too recent on {chain_id}, skipping exchange rate update.")
        self.telegram_bot.try_send_msg(
            chat_id=base_config.telegram.group_chat_id,
            text=f"Exchange rate update skipped on {chain_id}: Last update too recent.",
        )
        raise ExchangeRateError(
            f"Last update timestamp is too recent on {chain_id}, skipping exchange rate update."
        )
    //...
```

**Impact:** This issue has a low impact because `solve_withdrawal_queue_all_chains` currently calls `try_update_exchange_rate_all_chains` to update the exchange rates again. However, if, according to the comments, `solve_withdrawal_queue_all_chains` is designed to terminate upon failure to update the exchange rate, then this issue needs to be taken into consideration.

**Recommendation:** It is recommended not to let the exchange rate update failure of a single chain affect the updates of subsequent chains.

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22).

### [I-11] The on-chain exchange rate update failure will not terminate the WithdrawalSolver task

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/cea4d156cafd2d9a5757d21d85aaea1db81479d6/app/domain/boring_vault/service/boring_vault.py#L568)

**Description:**

When executing the WithdrawalSolver task, it will first attempt to update the on-chain exchange rate. According to the comments, if the on-chain exchange rate update fails, it is expected to throw an error and stop the task.

```python
def solve_withdrawal_queue_all_chains(self):
    """Solve a withdrawal requests."""
    # 1. Update exchange rates for all chains
    # If updaing exchange rate fails, it will raise an exception and stop the process.
    self.try_update_exchange_rate_all_chains()
    // ...
```

However, upon inspecting the `try_update_exchange_rate` function, it can be seen that when an error occurs during the on-chain update, the function simply returns instead of propagating the exception.

```python
def try_update_exchange_rate(self, chain_id: str, new_rate: int) -> str:
    """Try to update the exchange rate, catching any errors."""
    try:
        return self.update_exchange_rate(chain_id, new_rate)
    except Exception:
        return ""
```

**Impact:** The task continues to execute silently even after the on-chain exchange rate update fails, rather than stopping. This does not match the expected behavior.

**Recommendation:** It is recommended that call `update_exchange_rate` function raises an exception instead of `try_update_exchange_rate`. Or modify the comments to match the intended behavior.

**Status:** Fixed

**Client response:** Fixed in [PR-22](https://github.com/SwellNetwork/hlp-internal-be/pull/22).

### [I-12] Validity checks after price type conversion may fail to throw the expected error

**Files:** [`RedstoneStablecoinRateProvider.sol`](https://github.com/SwellNetwork/boring-vault/tree/ce21e49b7a7be06c3a96c3c3ea6982c32e3ff224/src/oracles/RedstoneStablecoinRateProvider.sol)

**Description:**

After fetching the price from the oracle, the price is first converted from `int256` to `uint256`, and only then is the `price > 0` check performed.

```solidity
function getRate() public view returns (uint256 rate) {
    //...
    rate = _calculateRate(_baseRate.toUint256(), _quoteRate.toUint256());

    _rateCheck(rate);
}

function _calculateRate(uint256 baseRate, uint256 quoteRate) internal view returns (uint256) {
    require(baseRate > 0, "Base rate must be greater than 0");
    require(quoteRate > 0, "Quote rate must be greater than 0");
    //...
}
```

Under the current integration with the Redstone oracle, this step poses no issue because Redstone includes an internal `price > 0` check when publishing prices.

However, in the future, the project may reuse with Chainlink. Chainlink may return a negative price in the event of an oracle failure. As a result, converting from `int256` to `uint256` before performing the `price > 0` check could directly trigger a `SafeCastOverflowedIntToUint` error, instead of the intended custom or expected error.

**Impact:** The protocol fails to throw the expected error in the event of an abnormal price.

**Recommendation:** It is recommended to check `price > 0` before performing the type conversion.

**Status:** Fixed

**Client response:** Fixed in [PR-6](https://github.com/SwellNetwork/boring-vault/pull/6/files).

### [I-13] Consider applying parameter validation

**Original severity:** Best Practices

**Files:** [`atomic_queue.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/contract/atomic_queue.py#L20)

**Description:**

The function lacks proper input validation and error handling for blockchain interactions. Specifically, `request.offer` and `request.user_address` are passed to the smart contract without verifying they are valid checksummed Ethereum addresses.

**Impact:** Invalid address formats may cause contract calls to fail.

**Recommendation:** Implement address validation using `Web3.is_checksum_address()` for both `request.offer` and `request.user_address` before contract interaction.

**Status:** Fixed

**Client response:** Fixed in [7b5fd1148af2fc90b76af58918300968a668eab2](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/7b5fd1148af2fc90b76af58918300968a668eab2).

### [I-14] Consider escaping data in send_mg function

**Original severity:** Best Practices

**Files:** [`boring_vault.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/service/boring_vault.py#L277)

**Description:**

Dynamic content sent to Telegram lacks proper output encoding, creating potential security risks when data is transmitted to external services. While the current risk is mitigated because the protocol team controls the data sources (smart contracts and internal calculations).

**Impact:** Implementing proper output encoding represents a security best practice that prevents potential injection vulnerabilities.

**Recommendation:** Implement HTML escaping for all dynamic content before transmission to Telegram using `html.escape()`. For example: `html.escape(str(self.boring_vault_address))`.

**Status:** Fixed

**Client response:** Fixed in [361d3ffb8a1dfb0f5517fc21b2f5ff385bc68dfd](https://github.com/SwellNetwork/hlp-internal-be/pull/13/commits/361d3ffb8a1dfb0f5517fc21b2f5ff385bc68dfd).

### [I-15] Consider refactoring textual SQL to ORM queries

**Original severity:** Best Practices

**Files:** [`atomic_request_repo.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/domain/boring_vault/repo/atomic_request_repo.py#L19)

**Description:**

The `atomic_request_repo.py` file inconsistently combines ORM and raw SQL usage. This practice is generally discouraged, as it can lead to reduced readability, maintainability, and consistency across the codebase. Whenever possible, ORM should be preferred over raw SQL.

An example of raw SQL usage is found in the `get_by_is_solved(...)` function:

```python
def get_by_is_solved(self, is_solved: bool) -> List[AtomicRequest]:
    stmt = text(
        """
        SELECT * FROM atomic_requests
        WHERE is_solved = :is_solved
        """
    )
    stmt = stmt.bindparams(bindparam("is_solved"))
    with Session(self.db_engine) as session:
        result = session.exec(
            statement=stmt,
            params={"is_solved": is_solved},
        )
        rows = result.all()
        return [AtomicRequest(**dict(row._mapping)) for r
```

**Impact:** There is no direct security impact. However, this is noted as a best-practice finding.

**Recommendation:** Refactor the function to use ORM queries instead of raw SQL.

**Status:** Acknowledged

**Client response:** Acknowledged but will keep current version, respect author’s coding style.

### [I-16] Missing oracle best-practice checks

**Original severity:** Best Practices

**Files:** [`oracle.py`](https://github.com/SwellNetwork/hlp-internal-be/blob/c650a499023489bfa5d6ac027b28a940a3c44cdc/app/shared/blockchain/oracle/oracle.py#L54)

**Description:**

The bot interacts directly with on-chain oracles to fetch price data used in various calculations. The current implementation of `oracle.py` includes a check for data staleness.

However, if the protocol team decides to switch from the current Redstone oracle to a provider like Chainlink, it is considered best practice to also verify that the returned answer is greater than 0.

Additionally, the current implementation does not consider sequencer downtime, which is relevant in some Layer 2 environments and should be checked to ensure oracle responses are valid and trustworthy.

**Impact:** These are best-practice recommendations and do not currently introduce a direct vulnerability.

**Recommendation:**

- Check that the returned oracle answer is greater than 0;
- Include a check for sequencer downtime;

**Status:** Fixed

**Client response:** Fixed in [83ea2a2c2a1c207ecc166449fc4fab73afef035a](https://github.com/SwellNetwork/hlp-internal-be/commit/83ea2a2c2a1c207ecc166449fc4fab73afef035a). For sequencer downtime checking, currently there’s no sequencer status checker for HyperEVM. Will update when we have it.
