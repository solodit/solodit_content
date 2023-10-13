**Auditors**

[Trust Security](https://twitter.com/trust__90)


---


# Findings


## Medium Risk
### TRST-M-1 Operator can spoof query fees to make net profit from pool

**Description:**
Operators call `collect()` to pay query fees to the indexer. The fees are accumulated for all allocations that end in the same epoch in a Rebates.Pool structure, later to be split per indexer using the Cobb-Douglas production function.
This fee structure should be resistant to query fee donations that:
⦁	Lower another indexer's total rebate amount
⦁	Result in a net positive for the donator (query fee MEV)
However, stress testing of the function found that guarantee 2 is not held. Operators, which are transitioning to be a decentralized role, can fake queries and increase their portion of the pool. This does not directly harm other indexers, because their share of the pool increases as well. However, the share of the rebate pool that stays in the protocol decreases.
An example is provided below. The rewards per indexer are calculated as follows:
 
𝑟𝑒𝑤𝑎𝑟𝑑(i) = totalRewards * (fee/totalFees)^𝛼 * (stake/totalStake)^1-𝛼 

Suppose 𝛼 = 0.4523, and the layout of pool participants is as follows:


|  Participant  | Fee  | Stake|
|---------------|------|------|
| Participant 1 | 511  | 515  |
| Participant 2 | 794  | 1    |


The calculated rewards are:
𝑟𝑒𝑤𝑎𝑟𝑑(1) = 1305 ∗ ( 511/1305 )^0.4523 ∗ ( 515/516 )^1−0.4523 ≅ 853

r𝑒𝑤𝑎𝑟𝑑(2) = 1305 ∗ ( 794/1305 )^0.4523 ∗ ( 1/516 )^1−0.4523 ≅ 34

Fees of the rebate pool not rewarded:
r𝑒𝑚𝑎𝑖𝑛𝑖𝑛𝑔 = 1305 − 853 − 34 = 418
At this point, participant 1 donates 720. The new layout is:

|  Participant  | Fee   | Stake|
|---------------|-------|------|
| Participant 1 | 1231  | 515  |
| Participant 2 | 794   | 1    |

The calculated rewards are:


r𝑒𝑤𝑎𝑟𝑑(1) = 2025 ∗ ( 1231/1305 )^0.4523 ∗ ( 515/516 )^1−0.4523 ≅ 1615

r𝑒𝑤𝑎𝑟𝑑(2) = 2025 ∗ ( 794/1305 )^0.4523 ∗ ( 1/516 )^1−0.4523 ≅ 43

Fees of the rebate pool not rewarded:
𝑟𝑒𝑚𝑎𝑖𝑛𝑖𝑛𝑔 = 2025 − 1615 − 43 = 367

Calculating pool loss (of profits) from donation:
𝑙𝑜𝑠𝑠 = 418 − 367 = 51

Profit was split between the participants:

𝑝𝑟𝑜𝑓𝑖𝑡(1) = 1615 − 720 − 853 = 42

𝑝𝑟𝑜𝑓𝑖𝑡(2) = 43 − 34 = 9

Attackers can donate query fees at any point before the allocation is finalized, which is when 
a certain number of epochs have passed since it was closed. This means in the final block 
there will be incentives for large amounts of MEV activity of the different participants, to the 
loss of the protocol.
A python script that fuzzes the Cobb-Douglas formula has been provided separately.


**Recommended Mitigation:**
Consider making use of a different awarding formula, which would disincentivize forging of 
query activities.

**Team Response:**
Acknowledged. There is ongoing economic research on alternatives to Cobb-Douglas, but for 
now we think this is an acceptable consequence of decentralizing gateways.



## Low Risk
### TRST-L-1 collected amount might be entirely burnt as protocol fees unintentionally

**Description:**
When `collect()` is called after allocation closed, entire pulled amount is consumed as protocol 
tax. The transition from closed state to finalized state is instantaneous at a specific epoch. 
Therefore, it may occur that money sent for curation and indexer query fees is consumed 
entirely as tax. This happens when the time between sending of TX and its execution is 
larger than the remaining dispute window.

**Recommended Mitigation:**
Consider adding an optional parameter in the collect() API, **allowMaxTax**. If users are willing 
for the fee to be completely burned, they may set it to true.

**Team Response:**
Acknowledged. The proposed fix would require updating the interface with the existing 
**AllocationExchange**, which is not upgradable, so instead we will document this risk in the 
function's notice so that callers ensure they call the function well before the end of the 
dispute window.

### TRST-L-2 Operator – Indexer trust assumptions
**Description:**
The fee collection mechanism in Graph Staking is still somewhat trusted. For example, if 
operator does not maintain sufficient GRT balance, the indexer would not be able to trigger 
collection from the operator. This could be seen as outside the scope of the Staking 
protocol; however it is Graph's responsibility to create an incentive structure that enables 
establishment of relations across the different Graph roles



## Informational
### Improve documentation
Documentation of the `collect()` function states:
```solidity
        /**
            * @dev Collect query fees from state channels and assign them to an allocation.
            * Funds received are only accepted from a valid sender.
            * To avoid reverting on the withdrawal from channel flow this 
         function will:
             * 1) Accept calls with zero tokens.
             * 2) Accept calls after an allocation passed the dispute period, in that case, all
                * the received tokens are burned.
                * @param _tokens Amount of tokens to collect
                * @param _allocationID Allocation where the tokens will be assigned
                */
```
Note that the highlighted text is no longer relevant, now that the operator is decentralized. 
It should be omitted.

### Redundant event emission

In `collect()`, the event AllocationCollected is emitted outside the main if block. It is 
recommended that it shall be placed inside the if block, as when queryFees is zero, the 
function doesn't change state and therefore shouldn’t emit an event.