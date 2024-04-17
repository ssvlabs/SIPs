|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Matheus Franco, Gal Rogozinski | Committee consensus          | Core       | open-for-discussion | 2024-03-05 |

## Summary

Aggregate `Attestation` and `Sync Committee` duties based on the committee of operators and the duties' slot. 

## Motivation

With the current design, a committee of operators associated with several validators may end up performing more than one attestation or sync committee duties on equivalent data. This proposal helps to decrease the number of messages exchanged in the network and the processing cost.

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

Under the new design we will have a `Committee` object that will be the top level object in charge of processing consensus messages and partial signature messages for the attestation and sync committee roles.

The `Committee` will hold a `CommitteeRunner` object for each slot it is running a beacon duty for.

For other duty roles the old design will remain.

#### Code

```go
// CommitteeDuty implements Duty
type CommitteeDuty struct {
	Slot         spec.Slot
	BeaconDuties []*BeaconDuty
}

// Committee is a committee of a unique set of operators that have shared validators
type Committee interface {
	// Initializes and starts the runner for the duties for the given slot
  	Start(duty Duty) error
	// ProcessMessage processes a message routed to this Committee
	ProcessMessage(msg *signedSSVMessage *types.SignedSSVMessage)
}

type Committee struct {
	Runners     map[spec.Slot]CommitteeRunner
	Shares      map[ValidatorPubkey]Share
	Network           Network
	Beacon            BeaconNode
	OperatorID        OperatorID
	QBFTParams        QBFTParams    
}

type QBFTParams interface {
    Operators()        []OperatorID
    Quorum()           uint64
    PartialQuorum() 	  uint64
}

type Share struct {
    // New field:
	// HighestAttestingSlot holds the highest slot for which attester duty ran for a validator
    HighestAttestingSlot spec.Slot
}

// CommitteeRunner manages the duty cycle for a certain slot
type CommitteeRunner interface {
	// Start the duty lifecycle for the given slot. Emits a message.
    StartDuty(duty Duty) error
	// Processes cosensus message
    ProcessConsensus(consensusMessage *qbft.Message) error
	// Processes a post-consensus message
    ProcessPostConsensus(msg PartialSignatureMessages) 
}

type CommitteeRunner struct {
	Shares          map[ValidatorPubkey]Share
    QBFTParams     QBFTParams
	QBFTController *qbft.Controller
	BeaconNetwork  *types.BeaconNetwork
}

// BeaconVote is the consensus data
type BeaconVote struct {
    BlockRoot         phase0.Root
    Source            phase0.Checkpoint
    Target            phase0.Checkpoint
}

// PartialSignatureMessage is a msg for partial Beacon chain related signatures (like partial attestation, block, randao sigs)
type PartialSignatureMessage struct {
	PartialSignature Signature `ssz-size:"96"` // The Beacon chain partial Signature for a duty
	SigningRoot      [32]byte  `ssz-size:"32"` // the root signed in PartialSignature
	ValidatorIndex           phase0.ValidatorIndex			// Changed from OperatorID
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

func ConstructSyncCommittee(vote BeaconVote, duty AttesterDuty) SyncCommitteeMessage{
    return SyncCommitteeMessage
    {
        Slot: duty.Slot,
        BlockRoot: vote.BlockRoot,
        ValidatorIndex: duty.ValidatorIndex,
    }
}
```
#### PartialSignatureMessages

The current structure that we have in code can be unchanged.

```go
type PartialSignatureMessages struct {
	Type     PartialSigMsgType
	Slot     phase0.Slot
	Messages []*PartialSignatureMessage
}

// PartialSignatureMessage is a msg for partial Beacon chain related signatures
type PartialSignatureMessage struct {
	PartialSignature Signature `ssz-size:"96"` // The Beacon chain partial Signature for a duty
	SigningRoot      [32]byte  `ssz-size:"32"` // the root signed in PartialSignature
    ValidatorIndex   types.ValidatorIndex
}
```

We must have the following flow:
1. When you `calculateExpectedRootsAndDomain` (similar to `sync_committee_aggregator`). Use the `duty` objects to reconstruct the proper beacon data objects (i.e. `Attestation`).
2. The above calculation can be used to create a mapping of `ValidatorIndex` to `root`
3. When processing the messages find all the roots that have quorums for a certain validator, mark them, and sumbit corresponding beacon data to beacon chain.
4. For the next message received attempt to complete quorum for other roots.


#### Happy Flow

1. `Committee` receives duties that match a certain slot. If the slot is higher then `highestDecidedSlot` starts consensus for the relevant roles, and initializes a `CommitteeRunner` for the relevant Validators.
2. `Committee` receives consensus messages and hands them over `CommitteeRunner` that hands them over to the `QBFTController` that has unchanged logic. The only difference is `BeaconVote` is used as the ConsensusData object.
3.  Once `ProcessConsensus` decides, the `Committee` will create a post consensus of PartialSignatureMessage that aggregates the beacon partial signature for all relevant validators.
4. `Committee` will process post-consensus partial signature messages and submit a beacon message for each validator.


### Stopping Runs

Previously we have letted new validator duties stop the run for the previous duty. For a committee this is not a good idea and instead we will count on a tuned `CutOffRound` per duty to stop the instance.

### Omitting Partial Signatures

If in post-consensus stage for attestation duty, the duty's slot is lower than a validator's `CommitteeShare's` `highestAttestingSlot`, the `CommitteeRunner` will omit the post-consesnus message for this validator. 

#### Sync Committee
`CutOffRound = 4  \\ one slot`

#### Attestation
`CutOffRound = 12 \\ one epoch`

### CommitteeID

An identifier for committee must be added to `MessageID`.

```go
type CommitteeID [32]byte

// Return a 32 bytes ID for the committee of operators
func getCommitteeID(committee []OperatorID) CommitteeID {
	// sort
	sort.Slice(committee, func(i, j int) bool {
		return committee[i] < committee[j]
	})
	// Convert to bytes
	bytes := make([]byte, len(committee)*4)
	for i, v := range committee {
		binary.LittleEndian.PutUint32(bytes[i*4:], uint32(v))
	}
	// Hash
	return CommitteeID(sha256.Sum256(bytes))
}
```

In order to route consensus messages to the correct consensus runner, the `CommitteeID` field will be included in the `MessageID` replacing `ValidatorPublicKey`.

### Role

Now a message can include data for both Attestation and SyncCommittee roles we will unify the `BeaconRole` for both.

```go
// List of roles
const (
	// Changed
	BNRoleAttesterOrSyncCommittee BeaconRole = iota
	BNRoleAggregator
	BNRoleProposer
	// BNRoleSyncCommittee is removed
	BNRoleSyncCommitteeContribution

	BNRoleValidatorRegistration
	BNRoleVoluntaryExit
)
```

### MessageID

`ValidatorPublicKey` and `CommitteeID` will be used interchangeably in `MessageID`. `Role` will change location because it is used to determine ID type.  `CommitteeID` is 32 bytes long so it will be encoded with a `0x00` 16 bytes prefix to make it the same length as `ValidatorPublicKey`.

```go
const (
	domainSize     = 4
	domainStartPos = 0
	roleTypeSize   = 4
	// CHANGE IN Positions
	roleTypeStartPos = domainStartPos + domainSize
	senderIDSize     = 48
	senderIDStartPos = roleTypeStartPos + roleTypeSize
)

// MessageID is used to identify and route messages to the right validator and Runner
type MessageID [56]byte

func (msg MessageID) GetDomain() DomainType {
	return msg[domainStartPos : domainStartPos+domainSize]
}

func (msg MessageID) GetRoleType() BeaconRole {
	roleByts := msg[roleTypeStartPos : roleTypeStartPos+roleTypeSize]
	return BeaconRole(binary.LittleEndian.Uint32(roleByts))
}

func (msg MessageID) GetSenderID() []byte {
	return msg[senderIDStartPos : senderIDStartPos+senderIDSize]
}

func parseMessageID(messageID MessageID) (DomainType, BeaconRole, []byte) {
	domain := messageID.GetDomain()
	role := messageID.GetRoleType()
	if role == BNRoleSyncCommitteeContribution {
		return domain, BNRoleAttesterOrSyncCommittee, messageID.GetSenderID()[16:]
	}
	return domain, role, messageID.GetSenderID()
}
```


### Consensus Data

We note that the data needed for a consensus execution is the same for all validators. Thus, we can create a `BeaconVote` object that will hold the data for all validators. The data needed for the `SyncCommittee` role is a subset of the data needed for the `Attestation` role. Thus we can always query the beacon node for attestation data and pass this data to relevant runners.

`ConsensusData` currently holds the `duty` field to make sure that all committee members agree on committee information. However, if there is a difference between committee members there is no guarantee that a run will be triggered on the first place. This is because different views may cause [different shuffles](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee). Therefore we can rely on local view of `duty`, and keep the field empty. 


## P2P

### Network Topology

Previously, to communicate on behalf of a certain validator, the operators computed the topic ID in the following way:

```go
topicID := hexToUint64(validatorPKHex[:10]) % subnetsCount
```

Now, the messages are related to a committee (not to a specific validator). Thus, if we were to use the committee's validators' associated topics, we would be sending multiple equal messages on the network. To avoid that, we will start to use a topic associated with the committee. It can be computed in a way similar to the following:

```go
 topicID := binary.LittleEndian.Uint64(getCommitteeID(committee)) % subnetsCount
```

The above computation is deterministic, so every operator can know in advance the correct topic to communicate. Plus, the hash function adds uniformity, so that the topics are evenly populated.

Notice that, even though non-aggregated duties (Proposal, Aggregator, Sync committee contribution, Validator registration, and Voluntary exit) don't gain improvements from the above method, they can also use it for consistency, with no drawbacks.

#### Drawbacks

- The uniformity of messages in topics is affected by two reasons:
 1. Since $|Committees| \leq |Validators|$, it's less probable that the uniform distribution of topic assignments is achieved (because we have fewer elements to distribute).
 2. In the previous version, the assignment of two validators to the same topic produced the same cost independently of the validators, e.g. each one's operators would need to listen to $\approx$ 1.02 more duties per epoch. Now, two committees assigned to the same topic produce a cost that is proportional to the committees' sizes. In other words, when two big committees (in terms of associated validators) collide in the hash%128 function, each will need to listen to many more messages compared to when two small committees collide. We rely on the uniformity property of the hash function to alleviate this hurdle.


### Message Validation

This duties transformation requires a propagation of changes in message validation.

Regarding syntax, the rules can stay the same.

Regarding semantics,
  - If `SSVMessage.MsgID` contains a CommitteeID (i.e. the role is attestation + sync committee) and if the CommitteeID doesn't exist in the current network, it should ignore the message.
  - If a `ValidatorIndex` in `SignedPartialSignatureMessage.Message.Messages` is incorrect, considering the ValidatorPublicKey or the CommitteeID in the MessageID, it should ignore the message.

Regarding duties' general rules,
  - The duplicated partial signature rule (same validator, slot and type but different signing root or signature) now applies only to the Proposal, Aggregate, Sync Committee Contribution, Validator Registration and Voluntary Exit duties (since attestation and sync committee partial signature messages may include the same signing root.)
  - If the same `ValidatorIndex` appears more than 2 times in `SignedPartialSignatureMessage.Message.Messages`, the message is rejected.

Regarding duties' specific rules,
  - The attestation's and sync committee's specific rules are dropped in favor of a common set of rules for the new duty type attestation + sync committee. Namely, the higher attestation limits are used (34 slots, 12 rounds).
  - We can no longer limit a single validator to two attestation attempts per epoch. If we were to count only attestation, we could limit a committee with $V$ validators to do $2 \times V$ consensus per epoch. However, the fact the sync committee duties use the same consensus instances as attestations can force us to tolerate 32 consensus instances from a committee in an epoch. Thus, we suggest limiting by $2 \times V$ executions only with a condition check that no validators of such committee are doing sync committee duties in the epoch.
  - For the `BNRoleAttesterOrSyncCommittee` role, the number of signatures in a `PartialSignatureMessages` is limited to $min(2 * V, V+SYNC\_COMMITTEE\_SIZE.)$.

Regarding implementation,
  - The `ConsensusState` is currently mapped by a `ConsensusID` that uses the validator public key and the role from the `MessageID`. Since `MessageID` will have a CommitteeID, instead of a validator public key, for attestations and sync committees, the mapping of `ConsensusState` will need to change. We suggest either changing `ConsensusID` to encompass also a `CommitteeID` or simply mapping `ConsensusState` by `MessageID`.

### GossipSub Scoring

The only two groups of the GossipSub scoring parameters that depend on the expected message rate are $P_2$ and $P_3$. $P_3$ is currently switched off, so no change needs to be done on that end. $P_2$ regards increasing a peer score for its "first delivery" messages and counts with three parameters:
 - A decay $d_2$
 - A cap value ($cap_2$) for the counter
 - A weight ($w_2$) to multiply the counter value and sum the result to the topic's score.

The decay $d_2$ can stay the same (0.3162277660168379, which decays 1 to 0.01 in 4 epochs). The cap value is computed using the mesh target size, $D = 8$, and the expected message rate per decay interval, and it's defined as the convergence value if one keeps sending twice the number of messages it's expected to send. In other words,

$$cap_2 = \frac{2m}{D} \times \frac{1}{(1-d_2)}$$

The message rate for a validator per decay interval (one epoch), used to be calculated as $600/10000 \times (32 \times 12)$. To account for all validators in a topic, we multiply this value by $\frac{V}{32}$. Then, to account for the improvement of this change, considering that the message rate will drop to 15% of the current value, we multiply it by $0.15$ and, finally, we get

$$cap_2 = \frac{2}{D} \times \left( \frac{600}{10000} \times (32 \times 12) \times \frac{V}{128} \times 0.15 \right) \times \frac{1}{1-d_2}$$

The weight depends on the cap value and the maximum score for $P_2$, defined as 80, and, thus,

$$w_2 = \frac{80}{cap_2}$$

#### Drawbacks

- This SIP makes the validator density in topics become less uniform and, thus, the message rate also becomes less uniform. Nonetheless, $P_2$ is a positive score component and is minimal in magnitude compared to the negative components. Thus, the message rate accuracy and consequent positive score fluctuations are not significant.

## Pre-requisites

- SIP #45