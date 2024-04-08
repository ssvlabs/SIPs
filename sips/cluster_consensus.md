|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Matheus Franco, Gal Rogozinski | Cluster consensus          | Core       | open-for-discussion | 2024-03-05 |

## Summary

Aggregate `Attestation` and `Sync Committee` duties based on the cluster of operators and the duties' slot. 

## Motivation

With the current design, a cluster of operators associated with several validators may end up performing more than one attestation or sync committee duties on equivalent data. This proposal helps to decrease the number of messages exchanged in the network and the processing cost.

## Rationale

The aggregation of duties is possible because the data that must be agreed on is independent of the validator.

For example, take a look at the `AttestationData` type.
```go
type AttestationData struct {
	Slot            Slot
	Index           CommitteeIndex
	BeaconBlockRoot Root `ssz-size:"32"`
	Source          *Checkpoint
	Target          *Checkpoint
}
```
The only validator-dependent field is `CommitteeIndex` and it does not have to be agreed on.

For the `Sync Committee` duty, operators agree on a `phase0.Root` data which is also independent of the validator.

## Improvement

According to Monte Carlo simulations using a dataset based on the Mainnet, this proposal reduces to $21.60$% the current number of messages exchanged in the network. Note that this result includes aggregating the post-consensus messages into a single message.

Regarding the number of bits exchanged, we estimate that this proposal will reduce the current value to, at least, $52.96$%. Notice that this reduction is not as significant as the number of messages reduction due to the larger post-consensus messages.

Again with Monte Carlo simulations using the Mainnet dataset, the number of attestation duties aggregated presented the following distribution.

<p align="center">
<img src="./images/cluster_consensus/aggregated_duties.png"  width="50%" height="10%">
</p>

## Spec changes

### Design

Under the new design we will have a `Cluster` object that will be the top level object in charge of processing consensus messages and partial signature messages for the attestation and sync committee roles.

The `Cluster` will hold a single `ConsensusRunner` and multiple `PartialSigRunners` for each Validator managed by the cluster.

For other duty roles the old design will remain.

#### Code

```go
// Cluster is a cluster of a unique set of operators that run the same validator set
type Cluster interface {
	// Initializes and starts the runner for the duties for the given slot
  	Start(slot spec.Slot, attesterDuties []*types.Duty, syncCommitteeDuties []*types.Duty) error
	// ProcessMessage processes a message routed to this Cluster
	ProcessMessage(msg *types.SSVMessage)
}

type Cluster struct {
	ClusterRunners     map[spec.Slot]ClusterRunner
	Network           Network
	Beacon            BeaconNode
	OperatorID        OperatorID
	ClusterShares     [spec.ValidatorPubKey]ClusterShares
}

// ClusterShare Partial Validator info needed for cluster duties
type ClusterShare interface {
	getSharePubKey()      []byte
	getCommittee()        []*Operator
	getQuorum()           uint64
	getPartialQuorum() 	  uint64
	getSigner()           BeaconSigner
	// highestAttestingSlot holds the highest slot for which attester duty ran for a validator
	highestAttestingSlot spec.Slot
}


// ClusterRunner manages the duty cycle for a certain slot
type ClusterRunner interface {
	// Start the duty lifecycle for the given slot. Emits a message.
    StartDuties(slot spec.Slot, attesterDuties []*types.Duty, syncCommitteeDuties []*types.Duty) error
	// Processes cosensus message
    ProcessConsensus(consensusMessage *qbft.Message) error
	// Processes a post-consensus message
	ProcessPostConsensus(msg ClusterSignaturesMessage) 
}

type ClusterRunner struct {
	Share          *types.Share
	QBFTController *qbft.Controller
	BeaconNetwork  *types.BeaconNetwork
	Signer		   *types.BeaconSigner
}

// BeaconVote is the consensus data
type BeaconVote struct {
    BlockRoot         phase0.Root
    Source            phase0.Checkpoint
    Target            phase0.Checkpoint
}

// PartialSignaturesMessage is the message that contains all signatures for each validator for each root
type PartialSignaturesMessage struct {
    ValidatorIndices []phase0.ValidatorIndex
    Roots []phase0.Root
    Signatures []phase0.BLSSignature
}

func ConstructAttestation(vote BeaconVote, duty AttesterDuty) Attestation {
    bits := bitfield.New(duty.CommitteeLength)
    bits.Set(duty.ValidatorCommitteeIndex)

    return Attestation{
        Slot: duty.Slot,
        BlockRoot: vote.BlockRoot,
        Source: vote.Source,
        Target: vote.Target,
        CommiteeIndex: duty.CommitteeIndex,
        AggregationBits: bits,
    }
}
```

#### Happy Flow

1. `Cluster` receives duties that match a certain slot. If the slot is higher then `highestDecidedSlot` starts consensus for the relevant roles, and initializes a `ClusterRunner` for the relevant Validators.
2. `Cluster` receives consensus messages and hands them over `ClusterRunner` that hands them over to the `QBFTController` that has unchanged logic. The only difference is `BeaconVote` is used as the ConsensusData object.
3.  Once `ProcessConsensus` decides, the `Cluster` will create a post consensus of PartialSignatureMessage that aggregates the beacon partial signature for all relevant validators.
4. `Cluster` will process post-consensus partial signature messages and submit a beacon message for each validator.

### Stopping Runs

Previously we have letted new validator duties stop the run for the previous duty. For a cluster this is not a good idea and instead we will count on a tuned `CutOffRound` per duty to stop the instance.

### Omitting Partial Signatures

If in post-consensus stage for attestation duty, the duty's slot is lower than a validator's `ClusterShare's` `highestAttestingSlot`, the `ClusterRunner` will omit the post-consesnus message for this validator. 

#### Sync Committee
`CutOffRound = 4  \\ one slot`

#### Attestation
`CutOffRound = 12 \\ one epoch`

### ClusterID

An identifier for cluster must be added to `MessageID`.

```go
[48]byte ClusterID

// Return a 48 bytes ID for the cluster of operators
func getClusterID(operatorIDs []OperatorID) ClusterID {
	// Create a 16 bytes constant prefix for cluters
	const prefix = []byte{0x00}

	// return the sha256 of the sortedIDs
	return ClusterID(prefix + sha256.Sum256(bytes(sorted(operatorIDs))))
}
```

In order to route consensus messages to the correct consensus runner, the `ClusterID` field will be included in the `MessageID` replacing `ValidatorPublicKey`.


#### Prefix Rationale

The 16 bytes prefix we are creating elongates the `ClusterID` to 48 bytes. The same length as `ValidatorPublicKey`. It may seem like a waste of space, but due to SSZ encoding it will actually save 16 bytes when compared to using a variable size array.

### MessageID

`ValidatorPublicKey` and `ClusterID` will be used interchangeably in `MessageID`. `Role` will change location because it is used to determine ID type.

```go
const (
	domainSize       = 4
	domainStartPos   = 0
	roleTypeSize     = 4
	// CHANGE IN Positions
	roleTypeStartPos =  domainStartPos + domainSize
	receiverIDSize   = 48
	receiverIDStartPos   = roleTypePos + roleTypeSize
)

// MessageID is used to identify and route messages to the right validator and Runner
type MessageID [56]byte

func (msg MessageID) GetDomain() []byte {
	return msg[domainStartPos : domainStartPos+domainSize]
}

func (msg MessageID) GetRoleType() BeaconRole {
	roleByts := msg[roleTypeStartPos : roleTypeStartPos+roleTypeSize]
	return BeaconRole(binary.LittleEndian.Uint32(roleByts))
}

func (msg MessageID) GetRecipientID() []byte {
	return msg[receiverIDStartPos : receiverIDStartPos+receiverIDSize]
}
```


### Consensus Data

We note that the data needed for a consensus execution is the same for all validators. Thus, we can create a `BeaconVote` object that will hold the data for all validators. The data needed for the `SyncCommittee` role is a subset of the data needed for the `Attestation` role. Thus we can always query the beacon node for attestation data and pass this data to relevant runners.

`ConsensusData` currently holds the `duty` field to make sure that all committee members agree on committee information. However, if there is a difference between committee members there is no guarantee that a run will be triggered on the first place. This is because different views may cause [different shuffles](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee). Therefore we can rely on local view of `duty`, and keep the field empty. 

### Consensus Message Validation

## P2P Message Validation

This duties transformation requires similar changes in message validation, namely:
  - Different consensus executions are tagged by the `MessageID`. This change would be propagated with no further issues. However, the `MessageID` is used to get the validator's public key and the duty's role which are used as an ID to store the consensus state. This must be changed to use the operators' committee and the duty's role, or even simply the `MessageID`.
- Message validation limits the number of attestation duties per validator by using the validator's public key contained in the `MessageID`. This is no longer possible. A new limitation can be accomplished by checking the number of validators a cluster of operators is assigned to. If this number is less than 32 (the number of slots in an epoch), then we can limit the number of attestation duties of such cluster per epoch. The only exception would be if such a cluster is assigned to a sync committee duty (considering that we will indeed merge attestations and sync committee duties altogether in the same consensus execution).

## Pre-requisites

- SIP #45
