| Author      | Title                                   | Category | Status              | Date       |
|-------------|-----------------------------------------|----------|---------------------|------------|
| Alan Li     | Merge commit and post consensus message | Core     | open-for-discussion | 2025-05-23 |

## Summary  
In proposer, aggregator, and sync committee aggregator duties, the post consensus phase requires operators to send partial signatures to peers on the agreed message from the QBFT consensus. This SIP proposes to merge the commit messages from the QBFT consensus and the post consensus messages into a single message.

## Motivation
The post consensus phase is consuming significant bandwidth and processing power in the network. 

## Design Overview
The post consensus phase can be made redundant if partial signatures are already communicated during the QBFT consensus. 

Note that the purpose of the consensus is still to decide on the same message (not the one with partial signatures) as it currently does, but only to append partial signatures in the commit phase communication.

## Security
Security and liveness of the consensus are not affected since we are not changing the protocol itself, we are only adding a new field in the commit message.

A quorum of partial signatures can be exchanged successfully under the same security assumptions as the current consensus protocol that the threshold of nodes are honest and active. 

## Specification

### 1. Allow QBFT instance to partial sign

Add a new field `beaconSigner` to the `Instance` struct to allow beacon partial signing.

```go
type Instance struct {
	State  *State
	config IConfig
	operatorSigner *types.OperatorSigner
    beaconSigner *types.BeaconSigner // New field to allow beacon partial signing

	processMsgF *types.ThreadSafeF
	startOnce   sync.Once
	forceStop  bool
	StartValue []byte
}

```

### 2. Include partial signatures in the commit message

Upon receiving prepare messages, the instance will sign the message with the `beaconSigner`.

```go
func (i *Instance) uponPrepare(msg *ProcessingMessage, prepareMsgContainer *MsgContainer) error {
    // ... 

	commitMsg, err := CreateCommit(i.State, i.operatorSigner, i.beaconSigner, proposedRoot)

    // ...
}

func CreateCommit(state *State, operatorSigner *types.OperatorSigner, beaconSigner *types.BeaconSigner, root [32]byte) (*types.SignedSSVMessage, error) {

	msg := &Message{
		MsgType:    CommitMsgType,
		Height:     state.Height,
		Round:      state.Round,
		Identifier: state.ID,

		Root: root,

        // New field to include partial signatures
        PartialSignatures: beaconSigner.Sign(root),
	}
	return Sign(msg, state.CommitteeMember.OperatorID, operatorSigner)
}
```

### 3. Aggregate partial signatures upon receiving commit messages

Upon receiving a quorum of commit messages, the instance will aggregate the partial signatures.

```go
func (i *Instance) UponCommit(msg *ProcessingMessage, commitMsgContainer *MsgContainer) (bool, []byte, *types.SignedSSVMessage, error) {

	// ... 

	if quorum {
        aggregatedPartialSignatures := aggregatePartialSignatures(commitMsgs)
	}

    // ...
}
```

### 4. Return the aggregated partial signatures to SSV

The aggregated partial signatures will be returned to SSV package to be sent to the beacon.

```go
// added new return value aggregatedBeaconSignature
func (i *Instance) ProcessMsg(msg *ProcessingMessage) (decided bool, decidedValue []byte, aggregatedCommit *types.SignedSSVMessage, aggregatedBeaconSignature []byte, err error) {
    
    // ...

	return i.State.Decided, i.State.DecidedValue, aggregatedCommit, aggregatedBeaconSignature, nil
}
```