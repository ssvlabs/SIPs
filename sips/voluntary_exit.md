| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| Alon Muroch | Voluntary Exit | Core     | Draft  |


**Summary**
Describes how zero coordination voluntary exit works.  
<em>Important to note that a user holding the validator’s private key can sign a voluntary exit as well.</em>


**Specification**

**Contract**  
Adds a new function accessible only by the validator's owner, pushes a voluntray exit event. 
Can only be called once.

_Why a contract interaction?_ Enable contract based applications to do contract->contract zero coordination exits.
Cost of such interaction is pretty minimal considering the value of a single validator.

**Networking**  
A dedicated subnet for voluntary exits will be added to the network for all exit related communication between operators.
All operators subscribe to the network.

**Messages**
New SSV MsgType is introduced

| Name       | Type    | Value | Description                          |
|------------|---------|-------|--------------------------------------|
| SSVVoluntaryExit | MsgType | 3     | SSVMessage type for voluntary exit messages |

The SSVMessage Data field will encode the voluntary exit SignedMessage data structure. 

```go
// VoluntaryExit provides information about a voluntary exit.
type VoluntaryExit struct {
    Epoch          uint64
    ValidatorIndex uint64
}

type ExitMessage struct {
    ValidatorPubKey ValidatorPK `ssz-size:"48"`
    Message         *VoluntaryExit
}


type SignedExitMessage struct {
    // Signature using the ethereum address that registered the validator
    Signature [65]byte `ssz-size:"65"`
    Message   ExitMessage
}
```
**New Runner**  
A new runner for voluntary exits will be created.

Upon exit message being parsed from EL, each node taking part in the specific cluster will broadcast SignedExitMessage including a partial signature for the voluntary exit.
Upon a quorum of partial signatures, each node will reconstruct a valid signature and broadcast it to the beacon chain


# Proposed solution

_Validator voluntary exit_ will be a new task that operators will need to perform. It'll be similar to validator registration with no consensus and no post-consensus steps (just pre-consensus).

For this, operators need to create a [`SignedVoluntaryExit`](https://github.com/attestantio/go-eth2-client/blob/b3d7ec9b3b1f9bda36f4853e48ca8ada2a7cdd91/spec/phase0/signedvoluntaryexit.go#L28):
```go
type SignedVoluntaryExit struct {
	Message   *VoluntaryExit
	Signature BLSSignature `ssz-size:"96"`
}
```
where _Signature_ is the validator's signature over [`VoluntaryExit`](https://github.com/attestantio/go-eth2-client/blob/master/spec/phase0/voluntaryexit.go#L27), defined as:
```go
type VoluntaryExit struct {
	Epoch          Epoch
	ValidatorIndex ValidatorIndex
}
type Epoch uint64
type ValidatorIndex uint64
```

This requires the following changes:
- New [_VoluntaryExitCall_ on BeaconNode](#beacon-node) interface.
- New [_PartialSigMsgType_](#partial-signature-message-type).
- New [_BeaconRole_](#beacon-role).
- New [Runner struct implementation](#runner) .



## Beacon Node

New call to be included to the `BeaconNode` interface (according to [attestantio/go-eth2-client:service.go](https://github.com/attestantio/go-eth2-client/blob/master/service.go#L383)).

```go
// ValidatorVoluntaryExitCalls interface has all validator voluntary exit duty specific calls
type ValidatorVoluntaryExitCalls interface {
    // SubmitVoluntaryExit submits a validator voluntary exit
    SubmitVoluntaryExit(voluntaryExit *phase0.SignedVoluntaryExit, sig phase0.BLSSignature) error
}

type BeaconNode interface {
	// GetBeaconNetwork returns the beacon network the node is on
	GetBeaconNetwork() types.BeaconNetwork
	AttesterCalls
	ProposerCalls
	AggregatorCalls
	SyncCommitteeCalls
	SyncCommitteeContributionCalls
	ValidatorRegistrationCalls
    ValidatorVoluntaryExitCalls
	DomainCalls
}
```




## Partial Signature Message Type

New _PartialSigMsgType_.

```go
type PartialSigMsgType uint64

const (
	// PostConsensusPartialSig is a partial signature over a decided duty (attestation data, block, etc)
	PostConsensusPartialSig PartialSigMsgType = iota
	// RandaoPartialSig is a partial signature over randao reveal
	RandaoPartialSig
	// SelectionProofPartialSig is a partial signature for aggregator selection proof
	SelectionProofPartialSig
	// ContributionProofs is the partial selection proofs for sync committee contributions (it's an array of sigs)
	ContributionProofs
	// ValidatorRegistrationPartialSig is a partial signature over a ValidatorRegistration object
	ValidatorRegistrationPartialSig
	// ValidatorVoluntaryExitPartialSig is a partial signature over a VoluntaryExit object
	ValidatorVoluntaryExitPartialSig
)
```

## Beacon Role

New _BeaconRole_.

```go
// BeaconRole type of the validator role for a specific duty
type BeaconRole uint64

// List of roles
const (
	BNRoleAttester BeaconRole = iota
	BNRoleAggregator
	BNRoleProposer
	BNRoleSyncCommittee
	BNRoleSyncCommitteeContribution

	BNRoleValidatorRegistration
    BNRoleValidatorVoluntaryExit
)
```



## Runner

New _ValidatorVoluntaryExitRunner_ implementation of the _Runner_ abstract structure.

Below, there are some functions of the new _ValidatorVoluntaryExitRunner_ runner to illustrate its behaviour.

```go
// Duty runner for validator voluntary exit duty
type ValidatorVoluntaryExitRunner struct {
	BaseRunner *BaseRunner

	beacon   BeaconNode
	network  Network
	signer   types.KeyManager
	valCheck qbft.ProposedValueCheckF

    voluntaryExit   *phase0.VoluntaryExit
}


// Validator voluntary exit duty doesn't need consensus nor post-consensus.
// It just performs pre-consensus with ValidatorVoluntaryExitPartialSig over
// a VoluntaryExit object to create a SignedVoluntaryExit 
func (r *ValidatorVoluntaryExitRunner) executeDuty(duty *types.Duty) error {
	voluntaryExit, err := r.calculateVoluntaryExit()
	if err != nil {
		return errors.Wrap(err, "could not calculate validator voluntary exit")
	}

	// get PartialSignatureMessage with voluntaryExit root and signature
	msg, err := r.BaseRunner.signBeaconObject(r, voluntaryExit, duty.Slot, types.DomainVoluntaryExit)
	if err != nil {
		return errors.Wrap(err, "could not sign VoluntaryExit object")
	}

	msgs := types.PartialSignatureMessages{
		Type:     types.ValidatorVoluntaryExitPartialSig,
		Slot:     duty.Slot,
		Messages: []*types.PartialSignatureMessage{msg},
	}

	// sign PartialSignatureMessages object
	signature, err := r.GetSigner().SignRoot(msgs, types.PartialSignatureType, r.GetShare().SharePubKey)
	if err != nil {
		return errors.Wrap(err, "could not sign randao msg")
	}
	signedPartialMsg := &types.SignedPartialSignatureMessage{
		Message:   msgs,
		Signature: signature,
		Signer:    r.GetShare().OperatorID,
	}

	// broadcast
	data, err := signedPartialMsg.Encode()
	if err != nil {
		return errors.Wrap(err, "failed to encode signedPartialMsg with VoluntaryExit")
	}
	msgToBroadcast := &types.SSVMessage{
		MsgType: types.SSVPartialSignatureMsgType,
		MsgID:   types.NewMsgID(r.GetShare().DomainType, r.GetShare().ValidatorPubKey, r.BaseRunner.BeaconRoleType),
		Data:    data,
	}
	if err := r.GetNetwork().Broadcast(msgToBroadcast); err != nil {
		return errors.Wrap(err, "can't broadcast signedPartialMsg with VoluntaryExit")
	}

    // stores value for later using in ProcessPreConsensus
    r.voluntaryExit = voluntaryExit

	return nil
}

// Returns *phase0.VoluntaryExit object with current epoch and own validator index
func (r *ValidatorVoluntaryExitRunner) calculateVoluntaryExit() (*phase0.VoluntaryExit, error) {
	epoch := r.BaseRunner.BeaconNetwork.EstimatedEpochAtSlot(r.BaseRunner.State.StartingDuty.Slot)
    validatorIndex := r.GetState().DecidedValue.Duty.ValidatorIndex
	return &phase0.VoluntaryExit{
        Epoch: epoch,
        ValidatorIndex: validatorIndex,
	}, nil
}

// Check for quorum of partial signatures over VoluntaryExit and,
// if has quorum, constructs SignedVoluntaryExit and submits to BeaconNode
func (r *ValidatorVoluntaryExitRunner) ProcessPreConsensus(signedMsg *types.SignedPartialSignatureMessage) error {
	quorum, roots, err := r.BaseRunner.basePreConsensusMsgProcessing(r, signedMsg)
	if err != nil {
		return errors.Wrap(err, "failed processing validator voluntary exit message")
	}

	// quorum returns true only once (first time quorum achieved)
	if !quorum {
		return nil
	}

	// only 1 root, verified in basePreConsensusMsgProcessing
	root := roots[0]
	fullSig, err := r.GetState().ReconstructBeaconSig(r.GetState().PreConsensusContainer, root, r.GetShare().ValidatorPubKey)
	if err != nil {
		return errors.Wrap(err, "could not reconstruct validator voluntary exit sig")
	}
	specSig := phase0.BLSSignature{}
	copy(specSig[:], fullSig)

    // create SignedVoluntaryExit using VoluntaryExit created on r.executeDuty() and reconstructed signature
    signedVoluntaryExit := &phase0.SignedVoluntaryExit{
        Message: r.voluntaryExit,
        Signature: specSig,
    }

    if err := r.beacon.SubmitVoluntaryExit(signedVoluntaryExit, specSig); err != nil {
        return errors.Wrap(err, "could not submit validator voluntary exit")
    }

	r.GetState().Finished = true
	return nil
}
```