| Author      | Title                    | Category | Status |
|-------------|--------------------------|----------|--------|
| Alon Muroch | Change SSV operators set | Core     | draft  |

**Overview**  
Changing the operator set for an SSV validator is the last resort in keeping the validator operational. Changing operators can happen for a number of reasons: change fee structure, replacing malicious/ low performing operator and more.  
This SIP describes how the SSV protocol can support operator set change.  

Changing an operator consists of:
1) Generating new shares
2) Triggering transfer validator
3) Syncing new operator set's slashing protection data
4) Transitioning to new operator set

**Generating new shares**  
New shares can be generated via a centralized dealer or DKG, the end result will be 3f+1 new shares for the new operator set/

**Triggering change set**  
Triggering a transfer validator happens by calling to below SSVNetwork contract function with the new operator set and shares  

```solidity
/**
     * @dev Transfers a validator.
     * @param publicKey Validator public key.
     * @param operatorIds new Operator ids(cluster) to transfer the validator to.
     * @param shares snappy compressed shares(a set of encrypted and public shares).
     * @param amount amount of tokens to be deposited for the validator's pod.
     * @param transitionEpoch indicates the beacon chain epoch in which the transition will happen
     */
    function transferValidator(
        bytes calldata publicKey,
        uint64[] memory operatorIds,
        bytes calldata shares,
        uint64 amount,
        uint64 transitionEpoch
    ) external;

    /**
         * @dev Emitted when validator was transferred between pods.
         * @param publicKey The public key of a validator.
         * @param podId The validator's new pod id.
         * @param shares snappy compressed shares(a set of encrypted and public shares).
         * @param transitionEpoch indicates the beacon chain epoch in which the transition will happen
         */
    event ValidatorTransferred(
        bytes publicKey,
        bytes32 podId,
        bytes shares,
        uint64 transitionEpoch
    );
```
_Notice the above function is for contracts V3_

<ins>transitionEpoch needs to be set at least 2 epochs into the future. Setting it lower might juprodize the validator</ins>

**Syncing new operator set's slashing protection data**  
Upon parsing a ValidatorTransferred event, each of the new operators will trigger a get highest decided sync from peers in the relevant subnet for each duty type (each one has it's own QBFT controller).  

Each highest decided will be validated as any other decided message is.

The highest decided beacon object will be saved in the slashing protection database

Upon parsing a ValidatorTransferred event, register to the relevant subnet to listen for any validator decided messages (to keep up to date with decdied duties)

**Transitioning to new operator set - new operators**  
Conditions before starting:
1) transitionEpoch reached
2) synced highest decided

Upon conditions meet, a new operator will start the next available duty (transitionEpoch+1).  
QBFT instance started will be synced highest decided + 1

**Transitioning to new operator set - old operators**  
* Upon first slot at transitionEpoch, stop any running duty execution (including running QBFT instances).
* Delete associated share from database
* Terminate all associated runners