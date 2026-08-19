**Auditors**

[JecikPo](https://x.com/jecikPo), [WarlordSam07](https://github.com/WarlordSam07)

**Source:** [CODESPECT audit report](https://github.com/CODESPECT-security/audit-reports/blob/main/067_Swell_ETH_STAKING_DEPOSIT_BOT.pdf)

# Findings

## High Risk

### [H-01] Deposit front-run of an aged key captures 32 ETH to an operator credential

**Files:** [`gates.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L97-L137), [`web3_chain.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L150-L162), [`plan.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/plan.py#L177-L246)

**Description:**

This off-chain bot decides which validator public keys receive a 32 ETH deposit funded from the pooled `DepositManager` balance. The contracts deliberately deny node operators any control over a validator’s withdrawal credential and always substitute the protocol’s own (the `DepositManager` address for swETH, a protocol-controlled `EigenPod` for rswETH), so on exit the staked ETH returns to the protocol. The `setupValidators` NatSpec in `IDepositManager` gives the off-chain service two front-running defenses: validate that each public key has not already been used for a validator setup, and snapshot the deposit contract’s `depositDataRoot` for re-check at broadcast. The bot performs only the snapshot (backed on-chain by `InvalidDepositDataRoot`) and never the used-key check, so the credential-fixing pre-deposit it is meant to catch goes unchecked. A malicious or compromised operator who holds a validator BLS key:

a. Registers key K with the `NodeOperatorRegistry` well in advance. The registry stores the pubkey and signature without verifying the signature (its own comment delegates this to an off-chain service, i.e. this bot). The registered signature is valid for the protocol credential, so K later passes the `bls_valid` gate;

b. Shortly before a run, submits a separate 1 ETH deposit to the beacon deposit contract for K, under a withdrawal credential the operator controls. Consensus fixes a validator’s credential on its first valid deposit, so K’s credential is now permanently the operator’s;

c. The bot later selects K (aged, registered, BLS-valid) and broadcasts the 32 ETH deposit. Consensus treats it as a top-up to an already-initialized validator and credits the 32 ETH to the operator’s credential, ignoring the protocol’s. Per `apply_deposit` in the consensus spec, the signature and withdrawal credential are used only for the first deposit to a pubkey; every later deposit only increases the balance;

The gates miss it. `key_age_mature` measures the time since K was added to the registry (`key_added_ts`, built from `OperatorAddedValidatorDetails` logs), not the time since the hostile pre-deposit, so registering K early clears the window with a full margin: [`gates.py:117-123`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L117-L123)

```python
now = ctx["finalized_ts"]
added = ctx["key_added_ts"]
min_age = ctx["min_key_age_secs"]
# @audit age anchored to registry-add time, not first on-chain deposit; aged key passes despite hostile pre-deposit
too_young = [pk for pk in ctx["pubkeys"] if now - added.get(pk, now) < min_age]
```

`not_on_beacon` reads the validator set at beacon HEAD, which a freshly pre-deposited key does not enter until the deposit is processed, so the pre-deposit is invisible during that lag; it never reads the deposit contract where the pre-deposit lands immediately. The `InvalidDepositDataRoot` guard only covers the short read-to-broadcast window, so a pre-deposit that settled earlier does not trigger it. No check asks whether a selected key already received a deposit, and to which credential.

**Impact:** Each front-run key permanently sends 32 ETH of pooled user funds to a validator whose withdrawal credential the operator controls, recoverable only by the operator on exit. A 1 ETH pre-deposit captures 32 ETH; the loss is direct, irrecoverable, repeatable across keys and runs, and applies to both swETH and rswETH. High if a malicious or compromised node operator is in scope; if operators are fully trusted, the front-run is an accepted risk under that assumption rather than a defect.

**Recommendation:** Before depositing, query the deposit-contract history (or a beacon source exposing prior/pending deposits) for each selected key and refuse the run if one exists under a credential other than the expected protocol or pod credential. Alternatives: anchor `key_age_mature` to the first on-chain deposit observed for the key rather than the registry-add time, or require operators to self-fund the full 32 ETH so there is no pooled top-up to capture. Separately, make `not_on_beacon` fail closed by distinguishing an empty beacon response from a complete one and cross-checking beacon finality against the execution finalized block.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). New `no_foreign_predeposit` gate: before depositing, the bot reads each selected key’s existing deposit credentials from the beacon node (Pectra `pending_deposits`) and halts the whole run if any key already carries a deposit under a credential other than its expected protocol/pod one. The finding’s "separately" recommendation — the fail-closed on-beacon read with an EL cross-check — is delivered under CODESPECT-03, and the same checks protect this read. The remaining dependency is the beacon node itself: the bot verifies it is synced, Pectra-aware, and ahead of the EL finalized block, and refuses on any doubt — but a node that lies within those checks is believed. Operator registration is governance-gated, which bounds who could attempt this.

**CODESPECT fix review:** Fixed (narrow residual). New `no_foreign_predeposit` gate halts on any key already deposited under a non-protocol credential; still reads at plan time, not re-checked at broadcast.

## Medium Risk

### [M-01] A single BLS-invalid front-of-queue key can halt allocated node-operator deposits

**Files:** [`gates.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L126-L137), [`plan.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/plan.py#L177-L184), [`web3_chain.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L144-L148)

**Description:**

Each run’s selected keys pass a fixed sequence of gates before broadcast. `bls_valid` refuses to broadcast if any selected key’s registry signature does not verify against its withdrawal credentials, which guarantees the bot never deposits to an unverifiable key. The problem is the scope: the refusal is global over the whole batch, so one invalid key blocks every operator selected in the same run. Three properties combine. First, the registry accepts a bad-signature key with no verification: `NodeOperatorRegistry.addNewValidatorDetails` stores (`pubKey`, `signature`) without checking the signature (delegated to this bot by design), so any live, enabled operator can register a key whose signature does not commit to the protocol or pod credential, with no special on-chain state required. Second, the bot folds the number of failed keys across all operators into a single count (`bls_unverified = len(resolved.unverified)`, `plan.py:199`) and the gate refuses the whole plan if it is non-zero, with no notion of which operator owns the offending key: [`gates.py:126-137`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L126-L137)

```python
def bls_valid(ctx):
    n_bad = ctx["bls_unverified_count"]
    # @audit batch-global refusal, no per-operator isolation; one bad key halts whole fleet
    if n_bad:
        return f"{n_bad} key(s) failed BLS verification / pod recovery"
    return True
```

Third, selection is FIFO front-of-queue (`selected[o.name] = pending[o.name][:want]`), and the contract consumes each operator’s keys strictly front-to-back: `usePubKeysForValidatorSetup` reverts `NextOperatorPubKeyMismatch` unless the next key consumed is the operator’s current head, so the bot cannot skip a bad head key. A blocked run deposits nothing and changes no on-chain state, so re-runs against the same snapshot re-select the same head key and fail again. The halt is self-perpetuating until the key is removed on-chain or the selected allocation changes, and under normal target-ratio allocation a live operator is allocated regularly, so this is a realistic operating condition rather than a theoretical edge case.

**Impact:** A single live node operator, malicious or merely careless with one key, can halt all allocated node-operator deposits protocol-wide (both swETH and rswETH) until a maintainer removes the offending key on-chain. Honest operators selected alongside it are blocked even though their own keys are valid. No funds are lost and no wrong-credential deposit occurs: the gate fires before broadcast, so the 32 ETH per validator stays in the `DepositManager` and nothing is signed. The block is recoverable through a routine on-chain key removal and is legible in the witness log as a clean `deposit.blocked` record at the `bls_valid` gate.

**Recommendation:** Fault-isolate by operator instead of refusing the whole batch. When an operator’s front-of-queue key fails `bls_valid`, allocate zero to that operator and proceed with the operators whose front prefixes verify, surfacing the offending operator and key as an incident in the witness log. Isolation must be per-operator (the registry consumes keys front-to-back, so a single bad key inside a prefix cannot be skipped), which stops one operator’s bad key from halting the whole fleet while still refusing to deposit any unverifiable key.

**Status:** Acknowledged

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). We considered the per-operator fault isolation and deliberately kept the batch-global halt: an unverifiable key from a vetted operator is an incident, and the right response is to stop and review out-of-band, not to isolate the operator and continue — the same whole-run-stop posture as CODESPECT-02. What we fixed is legibility: the refusal now names the offending operator, key, and cause — in the refusal itself and in the preview — so the review is immediately actionable.

**CODESPECT fix review:** Acknowledged. One bad key still halts the whole fleet; only the refusal message improved (names the operator, key, and cause).

### [M-02] Nonce read at latest not pending causes nonce collision on an in-flight transaction

**Files:** [`web3_broadcast.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_broadcast.py#L55-L66)

**Description:**

`Web3Broadcaster.prepare` builds the deposit transaction by hand and reads the signing account’s transaction count to set the nonce, with no `block_identifier`: [`web3_broadcast.py:64`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_broadcast.py#L64)

```python
# @audit no block_identifier defaults to "latest", ignores in-flight mempool tx, causing nonce collision
"nonce": self.w3.eth.get_transaction_count(self.account.address),
```

With no `block_identifier`, the call falls back to web3’s default block `"latest"`, which returns the count of mined transactions only and ignores transactions already signed and sitting in the mempool. If a previous deposit transaction was submitted but not yet mined, the count at `"latest"` is unchanged, so `get_transaction_count` returns the same nonce the in-flight transaction already occupies. The newly signed transaction takes the colliding nonce, and the node either rejects it (`replacement transaction underpriced`) or silently drops it, leaving the operator to manually cancel the original before a re-run can succeed. web3’s own `fill_nonce` reads `"pending"` precisely to include unmined transactions, but the bot hand-builds the transaction dict and never goes through it. The most realistic way to reach the stuck-transaction precondition is the bot’s static EIP-1559 fee cap: a base-fee spike above the fixed `maxFeePerGas` strands the first deposit transaction in the mempool, and the operator’s manual re-run then signs against the stale `"latest"` count and collides. The same condition arises if the signing key is used concurrently from a second process.

**Impact:** Denial of service and operational disruption, no fund loss. Dormant under single-run usage where `latest == pending`; it activates when a prior deposit transaction is stuck in the mempool (the condition the static fee cap creates) or when the signing key is used concurrently. In that state the freshly signed transaction collides on nonce with the in-flight one, and the operator must manually cancel the stuck transaction before re-running. No 32 ETH deposit is misdirected and no funds leave the `DepositManager`; the consequence is a blocked run requiring manual intervention.

**Recommendation:** Read the nonce at `"pending"` so the count includes unmined transactions:

```python
"nonce": self.w3.eth.get_transaction_count(self.account.address, block_identifier="pending"),
```

Pair this with a fee-bump / replacement path for the deposit transaction; the two deficiencies interact (a stuck transaction plus a stale-nonce re-run produces a collision with no automated recovery), and reading the pending nonce is a prerequisite for any EIP-1559 same-nonce replacement. Add a persistent single-run signer lock keyed by (`chain id`, `signer address`, `mode`) so two runs cannot sign concurrently, and refuse to execute if the witness log holds an unresolved `tx.signed` or `tx.in_flight_uncertain` hash not yet confirmed or reverted on chain.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). The nonce is read at `pending`, and the send path refuses to run while the signer has an in-flight transaction (`pending > latest`) — the stuck-tx re-run path that produced the collision. That check also covers the suggested refusal over the bot’s own record of signed transactions: reading the chain catches in-flight transactions signed outside this bot too. We did not add the signer lock: the bot is an interactively-confirmed, human-run CLI, so simultaneous runs from one signer are an operational error, and their collision fails clean on the node. We did not add fee-bump/replacement either: the bot sends one transaction and does not manage the mempool, so a stalled tx is an out-of-band operator step (see CODESPECT-10).

**CODESPECT fix review:** Fixed. The send path now aborts when `pending > latest`, so a stuck-tx re-run stops instead of colliding on nonce.

## Low Risk

### [L-01] Duplicate operator name silently shadows an operator

**Files:** [`config.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/config.py#L83-L86), [`plan.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/plan.py#L116-L122)

**Description:**

The bot allocates deposits by a greedy-deficit ratio scheme, and every allocation and selection dict is keyed by `operator.name`. `load_dataset` (`config.py:83-86`) applies no name-uniqueness check. Three planning dicts are built by name: [`plan.py:116-122`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/plan.py#L116-L122)

```python
# @audit dicts keyed by o.name; duplicate name silently overwrites, shadowing one operator to zero
active = {o.name: chain.active_count(o.address, block) for o in live}
pending = {o.name: chain.pending_keys(o.address, block) for o in live}
capacity = {name: len(keys) for name, keys in pending.items()}
```

When two operators share a name (with distinct addresses), the second overwrites the first in each dict, so the shadowed operator’s pending keys, active count, and capacity are replaced by the twin’s. The allocator computes slots over the collided maps, and the selection loop iterates all operators but writes `selected[o.name]` (`plan.py:177-184`), so the shadowed operator deposits zero keys while the surviving twin fills both slots. The batch stays internally consistent (the selection sums to `n`, all keys BLS-valid), so no gate detects the drift and no contract backstop fires. The shortfall witness is itself computed from the same collided dicts, so the off-ratio state is never surfaced. The duplicate-*address* case differs: two distinct names sharing one address select the same keys twice and are caught loudly by the `no_duplicate_pubkeys` gate.

**Impact:** Config integrity only. One legitimate operator gets zero deposits while its same-name twin is over-funded, so the validator fleet drifts off the protocol’s target allocation ratios silently, with no diagnostic surfaced. There is no fund loss and no wrong credential: the 32 ETH still reaches a legitimate operator’s valid keys under the correct withdrawal credential. The trigger requires a fully-trusted config author to duplicate a name in the dataset; there is no attacker-reachable path.

**Recommendation:** Assert name uniqueness in `load_dataset` (raise if `len(names) != len(set(names))`), or, more robustly, key all allocation and selection dicts by operator address, which is unique by construction and maps to on-chain operator identity, eliminating the shadowing class of bug at its root.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). `load_dataset` now rejects duplicate operator names or addresses (case-normalized) at load. We chose this over re-keying the maps by address: operators are identified by name everywhere the tool reports to a human, so names must be unique anyway — and once that’s enforced at load, the by-name maps are safe.

**CODESPECT fix review:** Fixed. `load_dataset` now rejects duplicate operator names and addresses, so the silent shadowing cannot enter.

### [L-02] No gate checks the operator enabled flag

**Files:** [`web3_chain.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L135-L142), [`plan.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/plan.py#L237-L245)

**Description:**

The on-chain `NodeOperatorRegistry` tracks an `enabled` boolean per operator and enforces it when consuming keys:

```solidity
// NodeOperatorRegistry.usePubKeysForValidatorSetup (L203-205)
if (!getOperatorForOperatorId[operatorId].enabled) revert CannotUseDisabledOperator();
```

The bot fetches the full operator tuple in `active_count` but reads only field index 4 (`activeValidators`) and discards field index 0 (`enabled`): [`web3_chain.py:135-142`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L135-L142)

```python
op = self._nor.functions.getOperatorForOperatorId(op_id).call(...)
# @audit op[0] enabled flag discarded; disabled operator passes all gates, reverts on-chain
return int(op[4]) # activeValidators -- op[0] (enabled) is discarded
```

None of the seven gates reads it. So a key belonging to an operator disabled on-chain between dataset curation and a run (for example, disabled by Swell governance) passes all seven gates: `plan.ok == True`, the preview prints `gates: PASS`, and the operator launches execute. The condition surfaces only at the `simulate` step, where `usePubKeysForValidatorSetup` reverts with a bare `CannotUseDisabledOperator`. Because simulate runs before the confirm prompt, the run aborts before the operator is asked go/no-go, and because the whole batch is one transaction, one disabled-operator key fails the entire batch.

**Impact:** Liveness / DoS only; no funds move, since simulate precedes broadcast and the revert is atomic. The operator gets an opaque `CannotUseDisabledOperator` revert rather than a legible gate refusal naming the offending operator, and must manually investigate which operator is disabled before re-running. The trigger is a Swell-side operator disabled on-chain after curation, or a dataset listing an already-disabled operator; there is no attacker-reachable path.

**Recommendation:** Read `op[0]` (already fetched in the same tuple) and add an eighth gate `operators_enabled` that refuses any plan containing a key from a disabled operator, naming the operator. Alternatively, filter disabled operators out of `live_operators()` during chain reads. Either approach replaces the bare simulate revert with a legible, pre-broadcast refusal.

**Status:** Fixed

**Client response:** Update from the client: Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). Of the two options we chose the blocking gate (`operators_eligible`), and made it dataset-level: it fires, naming the operator, even if the disabled operator contributed no keys this run — so a stale dataset entry is corrected rather than hidden until the run it happens to be selected in.

**CODESPECT fix review:** Fixed. New `operators_eligible` gate reads the on-chain enabled flag and blocks a disabled operator pre-broadcast instead of an opaque simulate revert.

### [L-03] Stale or rotated operator address crashes the run with an unhandled traceback

**Files:** [`web3_chain.py` (`pending_keys`)](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L144-L148), [`web3_chain.py` (`active_count`)](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L135-L142), [`cli.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/cli.py#L118-L150)

**Description:**

The bot reads each operator’s on-chain state through two paired calls. `active_count` resolves the operator id through the public mapping getter and tolerates an unknown address (`if op_id == 0: return 0`, since the getter never reverts). `pending_keys` has no such guard: [`web3_chain.py:144-148`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L144-L148)

```python
# @audit no op_id == 0 guard; reverts NoOperatorFound for a stale address
details = self._nor.functions.getOperatorsPendingValidatorDetails(addr).call(block_identifier=bid)
```

On-chain, `getOperatorsPendingValidatorDetails` routes through `_getOperatorIdSafe`, which reverts on the exact `op_id == 0` case `active_count` swallowed:

```solidity
// NodeOperatorRegistry.sol _getOperatorIdSafe
operatorId = getOperatorIdForAddress[_operatorAddress];
if (operatorId == 0) {
    revert NoOperatorFound(_operatorAddress);
}
```

The public mapping returns 0 for any deleted address, and `updateOperatorControllingAddress` deletes the old controlling address. So if an operator rotates their controlling wallet, or the dataset lists a typo’d, not-yet-registered, or since-removed address, `active_count(addr)` (`plan.py:116`) returns 0 gracefully but `pending_keys(addr)` (`plan.py:121`) reverts `NoOperatorFound`. web3 surfaces this as a `ContractLogicError`, which is not a `BroadcastError`; the `build_plan` call at `cli.py:120` is unwrapped, and the only handler (`cli.py:147`) catches `BroadcastError` alone, so the process dies with a raw traceback before any gate runs and with no `*.blocked` witness record. This differs from the disabled-operator case, which reaches a clean simulate revert and a witness record after the gates; here the crash happens during planning, before any gate.

**Impact:** Whole-run liveness DoS with the worst failure shape in the codebase: an unhandled traceback rather than a structured abort, and no witness record. A single stale or typo’d operator row takes down the entire run. No funds move; the 32 ETH per validator stays in the `DepositManager`. The trigger is a fully-trusted config author’s stale/typo’d dataset address or an on-chain rotation (`updateOperatorControllingAddress`) after curation; there is no attacker-reachable path. Applies to both products.

**Recommendation:** Make `pending_keys` mirror `active_count`: resolve the operator id first and return `[]` when `op_id == 0`, so a misconfigured or rotated operator simply contributes no keys and the existing shortfall and witness machinery records it legibly. Alternatively, pre-validate at load that every dataset operator address resolves to a nonzero on-chain operator id, turning the mid-planning crash into a legible refusal.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). `pending_keys` now mirrors `active_count` — resolving the operator id first and returning empty for an unresolved address — and the operator is then blocked, named, by the same `operators_eligible` gate as CODESPECT-04.

**CODESPECT fix review:** Fixed. A rotated or unresolved address now yields a clean gate block instead of a raw `NoOperatorFound` traceback.

### [L-04] The not_on_beacon gate fails open on an empty, lagging, or hostile beacon feed

**Files:** [`web3_chain.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L150-L162), [`gates.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L97-L102)

**Description:**

Gate 5 (`not_on_beacon`) is the bot’s primary check that a selected key has not already received a beacon-chain deposit. It is a plain set intersection of the selected pubkeys against a set built from the beacon REST response, where `data.get("data", [])` treats a missing array as empty: [`web3_chain.py:160-162`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L160-L162)

```python
with urllib.request.urlopen(req, timeout=30) as resp:
    data = json.load(resp)
# @audit missing/empty data array silently yields empty set; gate fails open
return {v["validator"]["pubkey"].lower() for v in data.get("data", [])}
```

A key absent from that set is treated as not-on-beacon, with no confirmation that the node is synced, canonical, serving a complete response, or even reachable, and the bot uses a single `--beacon` URL with no fallback or health check. So an empty body (`"data":[]`), a partial or MITM-injected response, or an HTTP 200 from a minority fork makes `beacon_known` empty or incomplete, and any selected key absent from it passes regardless of true on-chain state: absence of evidence is treated as evidence of absence. An empty `--beacon ""` value also passes argparse’s `required=True` while being falsy, silently disabling the gate, since `beacon_known` returns an empty set whenever `beacon_url` is falsy.

**Impact:** Standalone, a defense-in-depth weakening (liveness; the degraded-feed case needs no attacker). Its material risk is as an amplifier for the deposit front-run of an aged key: that attack relies on a window during which the attacker’s pre-deposit is not yet visible on the beacon validator set, and an unreliable or operator-controlled feed can extend that window across multiple runs. Low standalone.

**Recommendation:** Absence cannot be the integrity signal, because on a normal batch every selected key is legitimately not yet on beacon (`data == []`), so a naive `"refuse if data == []"` would block every deposit and still fail open on a partial response. Use out-of-band positive confirmation instead:

- Include a sentinel `id=<known-active-validator>` in the same query (the URL builder already joins arbitrary `id=` params) and fail closed if it is absent from `data[]`;

- Query `/eth/v1/node/syncing` and refuse unless `is_syncing == false`;

- Cross-check the beacon head slot against the execution-layer finalized block;

- Consider two independent beacon endpoints and block the run if they disagree;

Also reject an empty or whitespace-only `--beacon` value at startup.

**Status:** Fixed

**Client response:** Update from the client: Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). Implemented as recommended: an empty result is trusted only when the node reports synced, its head slot (converted to wall-clock time) is ahead of the EL finalized block, and a configured sentinel validator appears in the response; a transport error, a syncing node, a stale head, or a missing sentinel refuses. A beacon endpoint is now mandatory and an empty `--beacon` is rejected. We did not adopt the remaining "consider" item (two independent beacon endpoints) — a second trusted endpoint adds operational surface for little marginal benefit once freshness is cross-checked.

**CODESPECT fix review:** Fixed. `not_on_beacon` now fails closed unless the feed proves itself (synced, sentinel present, head ahead of EL finality); empty `--beacon` rejected.

### [L-05] verify_deposit(...) crashes instead of returning False on malformed-hex input

**Files:** [`bls.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/bls.py#L81)

**Description:**

`verify_deposit(...)` is the bot’s signature check on data from an untrusted RPC, and its docstring promises a fail-closed contract: malformed input returns `False`, never raises. It honours that for a wrong byte length (the `if len(...)` guard) and a malformed curve point (caught by `try/except ValueError`), but not for malformed hex, because the hex decode runs before the try:

```python
pk, wc, sig = _b(pubkey), _b(withdrawal_credentials), _b(signature) # outside the try
if len(pk) != 48 or len(wc) != 32 or len(sig) != 96:
    return False
# ...
try:
    return bool(PopSchemeMPL.verify(...))
except ValueError:
    return False
```

`_b(...)` calls `bytes.fromhex(...)`, which raises `ValueError` on an odd-length or non-hex string, so such input escapes `verify_deposit(...)` and crashes the run instead of returning `False`. This is fail-safe (a crash never returns a wrong `True`) and is not reachable through the normal pipeline (web3’s `_hex(...)` always emits even-length `0x...`); the issue is the violation of the documented "never raises" contract.

**Impact:** Fail-closed-by-crash: no wrong verification and no fund risk, but malformed input aborts the run with an uncaught exception instead of the documented `False`.

**Recommendation:** Move the `_b(...)` parsing inside the try (or wrap parse, length-check, and verify in one `try/except ValueError`) so every malformed input returns `False`.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). The hex parse moved inside the try, so malformed input returns `False` per the documented contract.

**CODESPECT fix review:** Fixed. The hex parse moved inside `try/except`, so malformed input returns `False` instead of crashing the run.

## Informational

### [I-01] Beacon validator query is an un-chunked GET with no error handling and can exceed strict URI limits (HTTP 414)

**Files:** [`web3_chain.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L150-L162)

**Description:**

`beacon_known` asks the beacon node which selected pubkeys are already known on the beacon chain (feeding the `not_on_beacon` gate). It concatenates every selected pubkey into one GET query string and issues a single un-chunked request with no error handling: [`web3_chain.py:154-161`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_chain.py#L154-L161)

```python
# @audit all pubkeys in one un-chunked GET, no try/except around the call; a transport error aborts the whole run
q = "&".join(f"id={urllib.parse.quote(pk)}" for pk in pubkeys)
url = f"{self.beacon_url.rstrip('/')}/eth/v1/beacon/states/head/validators?{q}"
```

Two issues. First, there is no `try/except` around the request, so any transport failure (a 414, a 5xx, a dropped connection) propagates out of `build_plan` and aborts the whole run, taking the `not_on_beacon` gate down with it. Second, the query length scales with the batch: each pubkey is 98 characters and contributes `id=` plus the key, so at the default `--cap 64` the request URL is about 6.6 KB. Common beacon-fronting proxies accept that at their defaults (nginx caps the request line at 8 KB; Apache’s `LimitRequestLine` is 8190 B), so it does not fail on a typical deployment. It returns HTTP 414 only on a stricter endpoint (for example an IIS-style front with `maxUrl 4096` / `maxQueryString 2048`) or once the cap is raised past roughly 78 keys, where the URL crosses 8 KB.

**Impact:** Availability and robustness only; no fund loss and no wrong deposit. On the common proxy defaults the default-cap batch is accepted, so this is a hardening item rather than a guaranteed failure. Where it does fire, the cause (URL length, not a beacon-data problem) is not obvious from the raised transport error, and the `not_on_beacon` gate cannot run for that batch. It is not attacker-triggerable and applies to both products, since the beacon read is shared.

**Recommendation:** Use the beacon API’s `POST /eth/v1/beacon/states/state_id/validators` form, which carries the validator IDs in the request body and is the spec’s intended mechanism for large ID lists, avoiding URI limits entirely. Alternatively, chunk the GET into batches of about 20 keys and union the responses. In either case, wrap the call so a transport failure fails the gate closed rather than aborting the whole run.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). The validator query now uses the beacon API’s POST form (ids in the body). The error-handling half of this finding is covered by the CODESPECT-03 fix, which fails the beacon read closed on any transport error.

**CODESPECT fix review:** Fixed. Beacon query is now a POST with ids in the body and wrapped to fail closed, so no HTTP 414 and no run-aborting transport error.

### [I-02] funding_covers_queue gate never fires on dead code

**Files:** [`gates.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/gates.py#L81)

**Description:**

`funding_covers_queue(...)` is one of the seven safety gates. Its stated job is to be an early, legible mirror of the `DepositManager`’s on-chain invariant, the contract reverts with `InsufficientETHBalance` unless the manager holds enough ETH to cover the new validators plus the ETH already owed to exiting validators. The gate is meant to catch that underfunding before any gas is spent. In the `funding_covers_queue`:

```python
need = ctx["n"] * (32 * 10**18) + ctx["exiting_eth_wei"]
have = ctx["deposit_manager_balance_wei"]
if have < need:
    return (
        f"underfunded: balance {have} < {ctx['n']}*32e18 + exitingETH "
        f"{ctx['exiting_eth_wei']} = {need}"
    )
```

The `if` condition will never be hit, because `n` is taken from the `have` and the `existingETH`, using the same 32-ETH constant, they are taken from the same source. Looks like nothing in the test suite tries to fire that gate, the only one aimed at underfunding is when `balance == exiting`.

**Impact:** Dead code. Gate does not capture the condition, yet it is already enforced by the correct calculation.

**Recommendation:** Remove the gate or reconsider when the gate would still make sense (e.g. if the values are re-read).

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). Removed the gate and relocated the invariant it was reaching for ("sizing never overspends the balance") into a unit test over a pure sizing function. We did not take the re-read alternative: a fresh-read funding check already exists in `simulate()` (`InsufficientETHBalance`) just before broadcast, so a re-armed gate would duplicate it.

**CODESPECT fix review:** Fixed. The tautological funding gate is gone; the invariant is now a tested pure function and `simulate()` still re-checks the balance at broadcast.

### [I-03] Hardcoded fee policy can strand the deposit tx

**Files:** [`web3_broadcast.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/web3_broadcast.py#L71)

**Description:**

`prepare(...)` builds and signs the run’s single deposit transaction and sets its EIP-1559 fees with a fixed policy and no bump/replace path:

```python
base = self.w3.eth.gas_price
txn["maxFeePerGas"] = base * 2
txn["maxPriorityFeePerGas"] = min(base, Web3.to_wei(2, "gwei"))
```

Despite the name, `base` is `eth.gas_price` (a suggested all-in gas price, not the EIP-1559 base fee), so `maxFeePerGas` is only 2x the current suggestion and the tip is capped at 2 gwei. These are read at `prepare(...)` time, after the human confirm, so the exposure is the prepare-to-inclusion window: if the base fee climbs past the 2x headroom (it can roughly double in 6 blocks under sustained congestion) or a flat 2 gwei tip is uncompetitive, the tx is underpriced and stalls unmined with no automatic bump. A stalled tx then feeds the in-flight re-run path above.

**Impact:** Liveness only; no fund loss, but the deposit transaction can stall in the mempool with no way to bump or replace it.

**Recommendation:** Make the fee policy configurable (or use a larger/dynamic headroom and a competitive tip), and add a bump-and-replace path for a stuck tx.

**Status:** Fixed

**Client response:** Addressed in [ede12615](https://github.com/SwellNetwork/eth-staking-deposit-bot/commit/ede12615c831e707413e020ec4dfb340f8637aab). Fees are now dynamic EIP-1559 read at send time — `maxFeePerGas = baseFee × headroom + node-suggested tip`, both configurable via CLI/env, with intentionally no hard ceiling (a ceiling would re-introduce exactly the stranding this finding describes). We did not add bump-and-replace: the bot sends one transaction and does not manage the mempool (consistent with CODESPECT-01), so a stalled tx is an out-of-band operator action.

**CODESPECT fix review:** Fixed. Dynamic EIP-1559 (real `baseFee`, configurable headroom and tip, no hard cap) replaces the old `base*2` / 2 gwei-cap that stranded the tx.

### [I-04] Operator can steer redistributed surplus by deepening its own queue

**Files:** [`allocate.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/allocate.py#L57), [`cli.py`](https://github.com/SwellNetwork/eth-staking-deposit-bot/blob/9298016aca651fef821aad0b9457dd9f4bdb8689/src/deposit_bot/cli.py#L69)

**Description:**

`allocate(...)` places each new validator with the operator furthest below its target ratio, converging the active fleet toward the configured ratios. When capacity (an operator’s pending-key count) is passed, an operator that cannot fill its share has its slots flow to operators that still have keys:

```python
def has_room(o: Operator) -> bool:
    return capacity is None or alloc[o.name] < capacity.get(o.name, 0)
# ...
candidates = [o for o in live if has_room(o)]
```

So an operator that keeps a deep, mature pending queue while peers stay shallow repeatedly absorbs the redistributed surplus, gaining active-validator share (and the rewards that follow it) above its target ratio. The drift is recorded and printed, but only as a non-blocking warning in `_print_preview(...)`; there is no concentration gate among the seven, so the deposit proceeds. It is self-correcting (an over-share operator carries a negative deficit until peers catch up), and `min_key_age` (12h) only rate-limits how fast a fresh queue can be built. No funds are at risk; absorbed validators still use the verified protocol/pod withdrawal credential.

**Impact:** Off-ratio concentration toward whichever operator keeps the deepest queue; economic and ratio drift only, no fund loss or wrong credentials.

**Recommendation:** Confirm with the protocol owner whether the concentration warning should become a blocking gate above some threshold rather than advisory auto-proceed.

**Status:** Acknowledged

**Client response:** Confirming per the recommendation: the concentration warning stays advisory. We do not read a deep queue as steering — supplying mature keys is exactly what operators are asked to do, and surplus redistributes only when peers have no keys to take their own share. A blocking gate would leave user capital idle to penalize the one operator that came prepared, enforcing a ratio that self-corrects anyway (an over-share operator is skipped until peers catch up). Key maturity rate-limits how fast a queue can be deepened, and the post-run share is shown in the preview before confirm.

**CODESPECT fix review:** Acknowledged (accepted). The concentration signal stays an advisory warning, no code change.
