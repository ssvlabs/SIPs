| Author      | Title                    | Category | Status |
|-------------|--------------------------|----------|--------|
| Alon Muroch | Change SSV operators set | Core     | draft  |


Open issues:
- Long range attack - past operator committee gets compromised, what it can do?
- historical syncing post committee change?

**Overview**  
Changing the operator set for an SSV validator is the last resort in keeping the validator operational. Changing operators can happen for a number of reasons: change fee structure, replacing malicious/ low performing operator and more.  
This SIP describes how the SSV protocol can support operator set change.  

Changing an operator consists of:
1) Generating new shares
2) Triggering transfer validator
3) Syncing new operator set's slashing protection data
4) Transitioning to new operator set

From the above there is a handoff of highest decided (including initial values for new operators slashing db) and a period of 2 epochs in which no operator set (old and new) execute duties.

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
     */
    function transferValidator(
        bytes calldata publicKey,
        uint64[] memory operatorIds,
        bytes calldata shares,
        uint64 amount,
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

**Syncing new operator set's slashing protection data**  
Upon parsing a ValidatorTransferred event, each of the new operators will trigger a get highest decided sync from peers in the relevant subnet for each duty type (each one has it's own QBFT controller).  

Each highest decided will be validated as any other decided message is.

The highest decided beacon object will be saved in the slashing protection database

Upon parsing a ValidatorTransferred event, register to the relevant subnet to listen for any validator decided messages (to keep up to date with decdied duties)

**Transitioning epoch**  
The epoch in which the transfer validator transaction is included is marked as the pre-transition epoch.  
Transition epoch = pre-transition epoch + 2

To let the new operator set sync and coordinate the transition we wait 2 epochs.  
TBD - fees calculation in contract

**Transitioning to new operator set - new operators**  
Conditions before starting:
1) transition epoch reached
2) synced highest decided

Upon conditions meet, a new operator will start the next available duty (in transition epoch).

**Transitioning to new operator set - old operators**  
* Upon pre-transition epoch, stop any running duty execution (including running QBFT instances).
* Delete associated share from database
* Terminate all associated runners