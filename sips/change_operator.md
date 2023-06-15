| Author      | Title                   | Category | Status   |
|-------------|-------------------------|----------|----------|
| Alon Muroch | Change operators set | Core     | rejected |

[Discussion] (https://github.com/bloxapp/SIPs/discussions/14)

## Overview  
Changing the operator set for an SSV validator is the last resort in keeping the validator operational. Changing operators can happen for a number of reasons: change fee structure, replacing malicious/ low performing operator and more.  
This SIP describes how the SSV protocol can support operator set change.  

Changing an operator consists of:
1) Generating new shares
2) Triggering transfer validator
3) Syncing new operator set's slashing protection data
4) Transitioning to new operator set

From the above there is a handoff of highest decided (including initial values for new operators slashing db) and a period of 2 epochs in which no operator set (old and new) execute duties.

## Generating new shares   
New shares can be generated via a centralized dealer or DKG, the end result will be 3f+1 new shares for the new operator set.
New shares must be randomly generated so to not be compatible with old shares. 

## Share Version
To distinguish between operator sets and enable historical message validation we add ShareVersion to SSVMessages.  
When a node validates a message, it should look at the share version and use the operator shares from it.

```go
type ShareVersion [32]byte

type SSVMessage struct {
    MsgType MsgType
    MsgID   MessageID
    Data    []byte
    Version ShareVersion
}

// ComputeShareVersion computes share version using solidity primitives keccak256(abi.encodePacked(operatorIds))
func ComputeShareVersion(share *Share) (ShareVersion, error) {
    uint64Solidity, _ := abi.NewType("uint64", "", nil)

    ret := ShareVersion{}

    arguments := abi.Arguments{}
    ids := make([]interface{}, 0)
    for _, operator := range share.Committee {
        arguments = append(arguments, abi.Argument{
            Type: uint64Solidity,
        })
        ids = append(ids, uint64(operator.OperatorID))
    }

    bytes, err := arguments.Pack(ids...)
    if err != nil {
        return ret, err
    }
    root := crypto.Keccak256(bytes)
    copy(ret[:], root)
    return ret, nil
}

```
## Transition
### Triggering change set
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
         */
    event ValidatorTransferred(
        bytes publicKey,
        bytes32 podId,
        bytes shares,
    );
```
_Notice the above function is for contracts V3_

### Syncing new operator set's slashing protection data 
Upon parsing a ValidatorTransferred event, each of the new operators will trigger a get highest decided sync from peers in the relevant subnet for each duty type (each one has it's own QBFT controller).  

Each highest decided will be validated as any other decided message is.

The highest decided beacon object will be saved in the slashing protection database

Upon parsing a ValidatorTransferred event, register to the relevant subnet to listen for any validator decided messages (to keep up to date with decdied duties)

### Transitioning epoch 
The epoch in which the transfer validator transaction is included is marked as the pre-transition epoch.  
Transition epoch = pre-transition epoch + 2

To let the new operator set sync and coordinate the transition we wait 2 epochs.  
TBD - fees calculation in contract

### Transitioning to new operator set - new operators 
Conditions before starting:
1) transition epoch reached
2) synced highest decided

Upon conditions meet, a new operator will start the next available duty (in transition epoch).

### Transitioning to new operator set - old operators  
* Upon pre-transition epoch, stop any running duty execution (including running QBFT instances).
* Delete associated share from database
* Terminate all associated runners

## Attack Vectors

<u>Signature mixing between old and new operator sets</u>  
Operator set <1,2,3,4> was changed, at height 101, to <1,2,3,5>.  
If operator 4 is malicious it will try to sign messages in epoch 101.  
Since key re-sharing requires new shares to be adopted, which are not compatible with the old set, no new decided message will be formed using any old set keys (as long as <f+1 malicious operators in both sets)

<u>Objective message verification</u>  
From an objective node trying to verify messages pre and post change it's important to know which message has which operator set.  
Using ShareVersion we can guarantee decided messages (considering <f+1 malicious) will always be signed with the up to date operator set

<u>Long range attacks</u>   
Replaced operators might see their shares as valueless as they are no longer taking part in operating the validator and not earning any rewards. Those operators might/ could be tempted to sell their old share keys to an attacker.  
It is important to note that changing the operator set frequently or replacing a significant amount of the original operators might compromise the validator.