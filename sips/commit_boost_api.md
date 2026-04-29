| Author | Title          | Category                             | Status  | Date               |
|-------|----------------|--------------------------------------|---------|--------------------|
| Alan Li | Commit-Boost Signing Support | Core | Open-for-discussion  | 2025-03-07 |

## Summary
This SIP describes the new commit-boost signing duty for SSV operators, requiring them to implement the commit-boost API, accepting requests to sign on some signing roots. This enhancement operates under the commit-boost framework and aims to support new features such as preconfirmation.

## Rationale
Commit-boost is becoming a standardized framework for Ethereum validators to interact with off-chain protocols such as preconfirmation protocols to increase profits for validators. 

By enabling commit-Boost API, this proposal also aligns with Ethereum’s roadmap toward better proposer-builder separation (PBS) while maintaining decentralization.

## Security Assumptions
1. When a signing request submits to a validator, all of its operators will receive the request. That is, they have the same commit boost module installed and the module forwards requests to all operators in a cluster.
2. Request sent to operators are assumed to be validated by external protocols. For example, for a preconfirmation request, the preconfirmation protocol makes sure that the request slot is valid, and we have not made too many commitments, etc. 

## Commit-Boost API
The [commit-boost API](https://commit-boost.github.io/commit-boost-client/api) is required to be implemented for validators to return the validator public key, receive signing request, and generate proxy key.

```go
type CommitBoostAPI interface {
    HandleGetPubkeys() ([]byte, error)
    HandleRequestSignature(keyType string, pubkey phase0.BLSPubKey, objectRoot phase0.Root) (phase0.BLSSignature, error)
    HandleGenerateProxyKey(pubkey phase0.BLSPubKey, scheme string) ([]byte, error)
}
```

`HandleGetPubkeys()` returns the validator's public key.

`HandleRequestSignature()` returns the signed object root under the [commit boost domain](https://github.com/Commit-Boost/commit-boost-client/blob/c5a16eec53b7e6ce0ee5c18295565f1a0aa6e389/crates/common/src/constants.rs#L3).

`HandleGenerateProxyKey()` returns a proxy key for a given proxy pubkey.

This SIP considers only `HandleRequestSignature()`, which introduces a new commit boost signing duty. `HandleGetPubkeys()` simply returns the public key of the validator, and `HandleGenerateProxyKey()` will be rejected.

## Specification
### CommitBoostPartialSignatureMsgType MsgType
The existing `msgID` used for route messages for other runners does not work for commit-boost signing runner, because the `msgID` assumes there is only one duty per slot per validator, which is not the case for commit-boost signing requests.

Due to this reason, a new message type `CommitBoostPartialSignatureMsgType` is introduced on the wire to identify `CBPartialSignatures` for commit-boost signing duty runners. 
```go
type CBPartialSignatures struct {
	RequestRoot phase0.Root `ssz-size:"32"`
	PartialSig  PartialSignatureMessages
}
```

The message contains a `PartialSignatureMessages` as other runners use, in addition, it contains the request root routing the message to the correct commit-boost signing duty runner. 


### Commit-Boost Signing Duty Runner
This duty handles commit boost signing requests sent from the commit-boost API. 

```go
type CBSigningRunner struct {
	BaseRunner *BaseRunner

	beacon         BeaconNode
	network        Network
	signer         types.BeaconSigner
	operatorSigner *types.OperatorSigner
	valCheck       qbft.ProposedValueCheckF

	requestRoot phase0.Root
	requestSig  chan phase0.BLSSignature
}
```

When a runner is initialized, the `requestRoot` is set. One runner is only created to process one signing request. Once the runner finishes the duty and has the complete signature, `requestSig` will be filled.

```go
func (r *CBSigningRunner) executeDuty(duty types.Duty) error {
	cbSigningDuty := types.CBSigningDuty{}
	if cb, ok := duty.(*types.CBSigningDuty); ok {
		cbSigningDuty = *cb
	} else if cb, ok := duty.(types.CBSigningDuty); ok {
		cbSigningDuty = cb
	} else {
		return errors.New("duty is not a CBSigningDuty")
	}
	request := cbSigningDuty.Request

	r.requestRoot = request.Root

	msg, err := r.BaseRunner.signBeaconObject(r, &cbSigningDuty.Duty, &request,
		duty.DutySlot(),
		types.DomainCommitBoost)
	if err != nil {
		return errors.Wrap(err, "failed signing attestation data")
	}

	preConsensusMsg := &types.PartialSignatureMessages{
		Type:     types.CBSigningPartialSig,
		Slot:     duty.DutySlot(),
		Messages: []*types.PartialSignatureMessage{msg},
	}

	CBPreConsensusMsg := &types.CBPartialSignatures{
		RequestRoot: r.requestRoot,
		PartialSig:  *preConsensusMsg,
	}

	msgID := types.NewMsgID(r.GetShare().DomainType, r.GetShare().ValidatorPubKey[:], r.BaseRunner.RunnerRoleType)

	encodedMsg, err := CBPreConsensusMsg.Encode()
	if err != nil {
		return err
	}

	ssvMsg := &types.SSVMessage{
		MsgType: types.CommitBoostPartialSignatureMsgType,
		MsgID:   msgID,
		Data:    encodedMsg,
	}

	sig, err := r.operatorSigner.SignSSVMessage(ssvMsg)
	if err != nil {
		return errors.Wrap(err, "could not sign SSVMessage")
	}

	msgToBroadcast := &types.SignedSSVMessage{
		Signatures:  [][]byte{sig},
		OperatorIDs: []types.OperatorID{r.operatorSigner.GetOperatorID()},
		SSVMessage:  ssvMsg,
	}

	if err := r.GetNetwork().Broadcast(msgToBroadcast.SSVMessage.GetID(), msgToBroadcast); err != nil {
		return errors.Wrap(err, "can't broadcast partial post consensus sig")
	}
	return nil
}
```

As shown above, when the signing root (request root + commit boost domain) is broadcasted to other operators, the request root is also included in the message, in order to route the message to the correct runner.

The runner uses the pre consensus phase to aggregate and return signed request.
```go
func (r *CBSigningRunner) ProcessPreConsensus(signedMsg *types.PartialSignatureMessages) error {
	quorum, roots, err := r.BaseRunner.basePreConsensusMsgProcessing(r, signedMsg)
	if err != nil {
		return errors.Wrap(err, "failed processing commit-boost signing message")
	}

	if !quorum {
		return nil
	}

	root := roots[0]
	fullSig, err := r.GetState().ReconstructBeaconSig(r.GetState().PreConsensusContainer, root, r.GetShare().ValidatorPubKey[:], r.GetShare().ValidatorIndex)
	if err != nil {
		// If the reconstructed signature verification failed, fall back to verifying each partial signature
		r.BaseRunner.FallBackAndVerifyEachSignature(r.GetState().PreConsensusContainer, root, r.GetShare().Committee,
			r.GetShare().ValidatorIndex)
		return errors.Wrap(err, "got pre-consensus quorum but it has invalid signatures")
	}
	specSig := phase0.BLSSignature{}
	copy(specSig[:], fullSig)

	r.requestSig <- specSig

	r.GetState().Finished = true
	return nil
}
```

### Validator Commit Boost: Manager for Preconfirmation Duties
Since the current `Validator` duty manager is designed to have only one runner per duty, it is not aligned with the multiple duties requirement for commit-boost signing. Therefore, a new `ValidatorCommitBoost` duty manager is created to manage commit-boost signing duty runners.

```go
type CBSigningRunners map[phase0.Root]Runner

type ValidatorCommitBoost struct {
	CBSigningRunners CBSigningRunners
	Network          Network
	CommitteeMember  *types.CommitteeMember
	Share            *types.Share
	Signer           types.BeaconSigner
	OperatorSigner   *types.OperatorSigner
}
```

The duty starts when a signing request is received at the operators' local commit boost module. All operators are expected to receive the same requests from the commit boost module at the same time. 

```go
func (v *ValidatorCommitBoost) HandleRequestSignature(keyType string, pubkey types.ValidatorPK, objectRoot phase0.Root) (phase0.BLSSignature, error) {
	// Proxy key is not supported currently
	if keyType != "consensus" {
		return phase0.BLSSignature{}, errors.New("invalid key type")
	}

	if pubkey != v.Share.ValidatorPubKey {
		return phase0.BLSSignature{}, errors.New("invalid pubkey")
	}

	var signingDuty = types.CBSigningDuty{
		Request: types.CBSigningRequest{
			Root: objectRoot,
		},
		Duty: types.ValidatorDuty{
			Slot:           v.BeaconNetwork.EstimatedCurrentSlot(),
			ValidatorIndex: v.Share.ValidatorIndex,
		},
	}

	err := v.StartDuty(signingDuty)
	if err != nil {
		return phase0.BLSSignature{}, errors.Wrap(err, "failed to start duty")
	}

	dutyRunner, exist := v.CBSigningRunners[objectRoot]
	if !exist {
		return phase0.BLSSignature{}, errors.Errorf("could not get duty runner for request %s", objectRoot.String())
	}
	sig := dutyRunner.GetSignature()

	return sig, nil
}
```

Unlike the `Validator` duty manager that routes duty to the an existing runner, the `ValidatorCommitBoost` duty manager creates a new commit-boost signing runner on receiving new signing request.

```go
func (v *ValidatorCommitBoost) StartDuty(duty types.CBSigningDuty) error {
	_, exist := v.CBSigningRunners[duty.Request.Root]
	if exist {
		return errors.Errorf("duty runner for request %s already exists", duty.Request.Root.String())
	}
	shareMap := make(map[phase0.ValidatorIndex]*types.Share)
	shareMap[v.Share.ValidatorIndex] = v.Share
	dutyRunner, err := NewCBSigningRunner(v.BeaconNetwork, shareMap, v.Beacon, v.Network, v.Signer, v.OperatorSigner)
	if err != nil {
		return errors.Wrap(err, "failed to create new commit-boost signing runner")
	}
	v.CBSigningRunners[duty.Request.Root] = dutyRunner
	return dutyRunner.StartNewDuty(duty, v.CommitteeMember.GetQuorum())
}
```

### Hard Rate Limit Constraint for Signature Requests
As a protection against spam requests, the number of commitments a validator can accept is capped at 1715 per slot.

This is due to the consideration that signature requests are most likely preconfirmation request for transaction inclusion, where one signing request corresponds to at least one transaction to include in the proposer duty. Given 36M gas limit per block and 21K minimum gas per transaction, 1715 is the upper bound number of signature requests a validator should receive. Note that in reality a validator should receive much less requests than this limit.

## Open Questions
- Should we enforce some rate limit on the signing requests?