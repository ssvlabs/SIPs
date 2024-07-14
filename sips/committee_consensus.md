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

### New Domain
This is a fork and new domains must be added:

```go
	AlanMainnet = DomainType{0x0, 0x0, MainnetNetworkID.Byte(), 0x1}
```

#### Code

```go
// Share holds all info about the validator share
// All the operator related data moved to operator
type Share struct {
	ValidatorIndex  phase0.ValidatorIndex
	ValidatorPubKey ValidatorPK      
	SharePubKey     ShareValidatorPK
	Committee       []ShareMember
	Quorum          uint64
	FeeRecipientAddress [20]byte
	Graffiti            []byte
}

// ShareMember holds ShareValidatorPK and ValidatorIndex
type ShareMember struct {
	SharePubKey ShareValidatorPK
	Signer      OperatorID
}

// CommitteeMember represents an SSV operator node that is part of a committee
type CommitteeMember struct {
	OperatorID        OperatorID
	CommitteeID         ssv.CommitteeID
	SSVOperatorPubKey []byte
	DomainType          DomainType
	Quorum, PartialQuorum uint64
	// All the members of the committee
	Committee []*Operator
}

// Operator represents all data in order to verify a operator's identity
type Operator struct {
	OperatorID        OperatorID
	SSVOperatorPubKey []byte
}

// Committee is a committee of a unique set of operators that have shared validators
type Committee struct {
	Runners                 map[spec.Slot]*CommitteeRunner
	CommitteeMember          types.CommitteeMember
	SignatureVerifier       types.SignatureVerifier
	CreateRunnerFn          func() *CommitteeRunner

	// Initializes and starts the runner for the duties for the given slot
  	StartDuty(duty CommitteeDuty) error
	// ProcessMessage processes a message routed to this Committee
	ProcessMessage(msg *types.SignedSSVMessage)
}

// CommitteeDuty aggregates attesting and sync committee duties
type CommitteeDuty struct {
	Slot         spec.Slot
	BeaconDuties []*BeaconDuty
}

// CommitteeRunner manages the duty cycle for a certain slot
type CommitteeRunner struct {
	// Important fields only
	Shares          map[ValidatorPubkey]Share
    CommitteeMember CommitteeMember
	QBFTController  *qbft.Controller
	BeaconNetwork   *types.BeaconNetwork

	// Start the duty lifecycle for the given slot. Emits a message.
    StartDuty(duty CommitteeDuty, quorum uint64) error
	// Processes cosensus message
    ProcessConsensus(consensusMessage *qbft.Message) error
	// Processes a post-consensus message
    ProcessPostConsensus(msg PartialSignatureMessages) 
}

// BeaconVote is the consensus data that is proposed by the QBFT leader for a CommitteeDuty
type BeaconVote struct {
    BlockRoot         phase0.Root
    Source            phase0.Checkpoint
    Target            phase0.Checkpoint
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

The current structure that we have in code can be kept with small changes.

```go
type PartialSignatureMessages struct {
	Type     PartialSigMsgType
	Slot     phase0.Slot
	Messages []*PartialSignatureMessage
}

// PartialSignatureMessage is a msg for partial Beacon chain related signatures (like partial attestation, block, randao sigs)
type PartialSignatureMessage struct {
	PartialSignature Signature `ssz-size:"96"` // The Beacon chain partial Signature for a duty
	SigningRoot      [32]byte  `ssz-size:"32"` // the root signed in PartialSignature
	Signer         OperatorID
	ValidatorIndex phase0.ValidatorIndex	// addition
}
```

We must have the following flow:
1. Calculate `expectedRootsAndBeaconObjects()`. Use the `duty` objects to reconstruct the proper beacon data objects (i.e. `Attestation`).
2. The above calculation can be used to create a mapping of `ValidatorIndex` to `root`, and `root` to `beaconObject`.
3. When processing the messages find all the roots that have quorums for a certain validator, mark them, and submit corresponding beacon data to the beacon chain.
4. For the next message received attempt to complete quorum for other roots.

For the `CommitteeRunner`, we will have a different validation of partial signature messages than for other runners. Runners like `ProposerRunner` and `AggregatorRunner` can validate the `SigningRoot` since they know exactly its expected value. For the `CommitteeRunner`, the number of roots depends on the number of validators that the committee has. Such a state may be divergent between operators because we assume no synchronization on validator sets. Thus, we limit ourselves to the standard validation (e.g. correct identifier, the maximum possible number of roots and so on).

#### Happy Flow

1. `Committee` receives duties that match a certain slot. Then `StartDuty` and initialize a `CommitteeRunner` for the relevant Validators and slot.

```go
// StartDuty starts a new duty for the given slot
func (c *Committee) StartDuty(duty *types.CommitteeDuty) error {
	if _, exists := c.Runners[duty.Slot]; exists {
		return errors.New(fmt.Sprintf("CommitteeRunner for slot %d already exists", duty.Slot))
	}
	c.Runners[duty.Slot] = c.CreateRunnerFn()
	return c.Runners[duty.Slot].StartNewDuty(duty)
}
```

2. `Committee` receives consensus messages and hands them over `CommitteeRunner` that hands them over to the `QBFTController` that has unchanged logic. The only difference is `BeaconVote` is used as the ConsensusData object.
3.  Once `ProcessConsensus` decides, the `Committee` will create a post consensus of PartialSignatureMessage that aggregates the beacon partial signature for all relevant validators.
4. `Committee` will process post-consensus partial signature messages and submit a beacon message for each validator.

### Cutoff Round
Previously new duties stopped consensus intsances. We cannot do that anymore since duties don't neccesarily map to specific validators. So we create a better Cutoff Round that will stop the consensus instance in a more sensible time. If the Sync Committee beacon message is longer than a slot tehn it won't enter the beacon chain.
An Attestation has at most 2 epochs to be included. However since after round 8 we use a slow timeout, if after 4 additional change rounds the instance didn't decide then the chances of it deciding are very low.

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

This is used to route the message to the correct runner. Since `CommitteeRunner` has several beacone roles we will create a new `RunnerRole` in the MessageID.

```go
type RunnerRole int32

const (
	RoleCommittee RunnerRole = iota
	RoleAggregator
	RoleProposer
	RoleSyncCommitteeContribution

	RoleValidatorRegistration
	RoleVoluntaryExit

	RoleUnknown = -1
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

func (msg MessageID) GetRoleType() RunnerRole {
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

Up until now, `ConsensusData` helds the `duty` field to make sure that all committee members agree on committee information. However, if there is a difference between committee members there is no guarantee that a run will be triggered on the first place. This is because different views may cause [different shuffles](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee). Therefore we can rely on local view of `duty`, and keep the field empty. 



### Value Check

Now that consensus handles value for multiple validators there is the issue that a value may be considered to be slashable by some validators but not by others. This can happen if some operators have different views of the ethereum chain. During the consensus value-check process we check whether a value is considered slashable by at least one of the validators. If so the check fails. Since we use one value for all validators, the `CommitteeIndex` takes a value of `MaxUInt64` to indicate this ambiguous state.

```go
func BeaconVoteValueCheckF(
	signer types.BeaconSigner,
	slot phase0.Slot,
	sharePublicKeys []types.ShareValidatorPK,
	estimatedCurrentEpoch phase0.Epoch,
) qbft.ProposedValueCheckF {
	return func(data []byte) error {
		bv := types.BeaconVote{}
		if err := bv.Decode(data); err != nil {
			return errors.Wrap(err, "failed decoding beacon vote")
		}

		if bv.Target.Epoch > estimatedCurrentEpoch+1 {
			return errors.New("attestation data target epoch is into far future")
		}

		if bv.Source.Epoch >= bv.Target.Epoch {
			return errors.New("attestation data source >= target")
		}

		attestationData := &phase0.AttestationData{
			Slot: slot,
			// Consensus data is unaware of CommitteeIndex
			// We use MaxUInt64 to not run into issues with the duplicate value slashing check:
			// (data_1 != data_2 and data_1.target.epoch == data_2.target.epoch)
			Index:           math.MaxUInt64,
			BeaconBlockRoot: bv.BlockRoot,
			Source:          bv.Source,
			Target:          bv.Target,
		}

		for _, sharePublicKey := range sharePublicKeys {
			if err := signer.IsAttestationSlashable(sharePublicKey, attestationData); err != nil {
				return errors.Wrap(err, "slashable attestation")
			}
		}
		return nil
	}
}
```

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
  - For the `RoleCommittee` role, the number of signatures in a `PartialSignatureMessages` is limited to $min(2 * V, V+SYNC\_COMMITTEE\_SIZE.)$.

Regarding implementation,
  - The `ConsensusState` is currently mapped by a `ConsensusID` that uses the validator public key and the role from the `MessageID`. Since `MessageID` will have a CommitteeID, instead of a validator public key, for attestations and sync committees, the mapping of `ConsensusState` will need to change. We suggest either changing `ConsensusID` to encompass also a `CommitteeID` or simply mapping `ConsensusState` by `MessageID`.

### GossipSub Scoring

The only two groups of the GossipSub scoring parameters that depend on the expected message rate are $P_2$ and $P_3$. $P_3$ is currently switched off, so no change needs to be done on that end. $P_2$ regards increasing a peer score for its "first delivery" messages and counts with three parameters:
 - A decay $d_2$
 - A cap value ($cap_2$) for the counter
 - A weight ($w_2$) to multiply the counter value and sum the result to the topic's score.

The decay $d_2$ can stay the same (0.3162277660168379, which decays 1 to 0.01 in 4 epochs). The cap value is defined as the convergence value if one keeps sending twice the number of messages it's expected to send. Thus, it's computed using the mesh target size, $D = 8$, and the expected message rate per decay interval. In other words,

$$cap_2 = \frac{2m}{D} \times \frac{1}{(1-d_2)}$$

The message rate for a validator per decay interval (one epoch), used to be calculated as $600/10000 \times (32 \times 12)$. To account for all validators in a topic, we used to multiply this value by $\frac{V}{128}$. However, now, the number of validators in a topic is not uniform (so we can't use the $\frac{V}{128}$ estimation) and the message rate should be regarded as "by committee" instead of "by validator" since the number of messages depends on the aggregation of the committee's validators duties.

Therefore, to compute the message rate for a certain topic, we need to get, as input, the list of committees and validators present in such a topic and adjust the calculation as the following script:

```go
// Expected number of committee duties per epoch due to attestation beacon duties given
// the number of validators in the committee
func expectedNumberOfCommitteeDutiesPerEpochDueToAttestation(numValidators int) float64 {
	k := float64(numValidators) // number of validators
	n := 32.0 // slots
	// Probability that all validators are not assigned to slot i
	probabilityAllNotOnSlotI := math.Pow((n-1)/n, k)
	// Probability that at least one validator is assigned to slot i
	probabilityAtLeastOneOnSlotI := 1 - probabilityAllNotOnSlotI
	// Expected value for duty existence ({0,1}) on slot i
	expectedDutyExistenceOnSlotI := 0 * probabilityAllNotOnSlotI + 1 * probabilityAtLeastOneOnSlotI
	// Expected number of duties per epoch
	expectedNumberOfDutiesPerEpoch := n * expectedDutyExistenceOnSlotI

	return expectedNumberOfDutiesPerEpoch
}

// Expected number of committee duties per epoch due to only
// sync committee beacon duties (no attestations) given the
// number of validators in the committee
func expectedNumberOfSingleSCCommitteeDutiesPerEpoch(numValidators int) float64 {
	// Probability that a validator is not in sync committee
	chanceOfNotBeingInSyncCommittee := 1 - syncCommitteeProbability()
	// Probability that all validators are not in sync committee
	chanceThatAllValidatorsAreNotInSyncCommittee := math.Pow(chanceOfNotBeingInSyncCommittee, float64(numValidators))
	// Probability that at least one validator is in sync committee
	chanceOfAtLeastOneValidatorBeingInSyncCommittee := 1 - chanceThatAllValidatorsAreNotInSyncCommittee

	// Number of committee duties if at least one validator is in sync committee
	committeeDutiesPerEpochIfOneValidatorIsInSyncCommittee := 32

	// Expected number of committee duties due to attestation
	expectedCommitteeDutiesPerEpochDueToAttestation := expectedNumberOfCommitteeDutiesPerEpochDueToAttestation(numValidators)

	// Number of committee duties per epoch created due to only sync committee duties
	expectedCommitteeDutiesDueToOnlySyncCommittee := committeeDutiesPerEpochIfOneValidatorIsInSyncCommittee - expectedCommitteeDutiesPerEpochDueToAttestation

	// Expected number of committee duties per epoch created due to only sync committee duties
	return chanceOfAtLeastOneValidatorBeingInSyncCommittee * expectedSlotsWithNoDuty
}

// Main: Expected message rate per epoch for a topic given the map: Committee -> Number of validators
func expectedEpochMessageRate(committeeToNumberOfValidatorsMap map[Committee]int) float64 {
	totalEpochMessageRate := float64(0)

	for committee, numValidators := range committeeToNumberOfValidatorsMap {
		committeeSize := committee.NumberOfOperators()

		// Attestation
		totalEpochMessageRate += expectedNumberOfCommitteeDutiesPerEpochDueToAttestation(numValidators) * numMessagesForDutyWithoutPreConsensus(committeeSize)

		// Sync committee
		totalEpochMessageRate += expectedNumberOfSingleSCCommitteeDutiesPerEpoch(numValidators) * numMessagesForDutyWithoutPreConsensus(committeeSize)

		// Aggregator
		totalEpochMessageRate += float64(numValidators) * aggregatorProbability() * numMessagesForDutyWithPreConsensus(committeeSize)

		// Proposal
		totalEpochMessageRate += float64(numValidators) * 32 * proposalProbability() * numMessagesForDutyWithPreConsensus(committeeSize)

		// Sync committee aggregator
		totalEpochMessageRate += float64(numValidators) * 32 * syncCommitteeAggregatorProbability() * numMessagesForDutyWithPreConsensus(committeeSize)
	}

	return totalEpochMessageRate
}
```

The above function will return the expected epoch message rate for the topic, $m$, which allows us to compute $cap_2$.

The weight, $w_2$, depends on the cap value and the maximum score for $P_2$ (defined as 80). It's computed as:

$$w_2 = \frac{80}{cap_2}$$

#### Drawbacks

- This SIP makes the validator density in topics become less uniform and, thus, the message rate also becomes less uniform. Nonetheless, $P_2$ is a positive score component and is minimal in magnitude compared to the negative components. Thus, the message rate accuracy and consequent positive score fluctuations are not significant.

## Pre-requisites

- SIP #45
