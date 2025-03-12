| Author | Title          | Category                             | Status  | Date               |
|-------|----------------|--------------------------------------|---------|--------------------|
| Alan Li | Preconfirmation Support | Core | Open-for-discussion  | 2025-03-07 |

## Summary
This SIP describes new preconfirmation duty for SSV operators, requiring them to sign, aggregate, and submit preconfirmation requests to a preconfirmation protocol sidecar. This enhancement operates under the commit-boost framework and aims to support different preconfirmation protocol implementations.

## Rationale
This proposal aligns with Ethereum’s roadmap toward better proposer-builder separation (PBS) while maintaining decentralization. Instead of relying on centralized services for preconfirmation, SSV operators can collectively perform this duty in a trust-minimized manner. 

For validators on SSV network, supporting preconfirmation also potentially increase validators' profits.

## Background
There are two types of preconfirmation protocol design:
1. A sidecar forwards preconf requests to the validator, the validator signs the requests and send back to the sidecar, which handles communication with relays for building blocks with the signed constraints. ETHGas is an example protocol for this type.
2. The validator delegates preconf request signing to relays, the relays handle preconf signing and block building, finally return blocks that satisfy constraints through the preconf sidecar to the validators for submission. Primev is an example protocol for this type.

For design type 2, SSV network doesn't have to change anything because everything is delegated to the relays. This proposal aims to support protocols with design type 1.

## Assumptions
1. Validators only register to preconf services that they trust, spam requests are not in the scope of this SIP. 
2. When a valid preconf request submits to a validator, all of its operators' commit boost instances will receive the request. 
3. The preconf sidecar handles request validation. Only valid preconf requests creates `PreconfCommitment` duty. For example, the request slot is valid, and we have not made too many commitments, etc. 
4. For now we assume all operators have the same commit boost module installed (i.e. run the same preconf protocol). 
5. The relays handles block building. It returns valid block based on the signed preconf requests. Every operator talks to the same relays. 
6. We trust the blinded blocks proposed by the leader of the proposer duty.

## Specification
### Commit-Boost API
The commit-boost API allows validators to sign and submit preconf request root, and get constrained blocks from relays.

```go
type CommitBoostAPI interface {
    GetNewRequest() (requestRoot phase0.Root, error)
    SubmitCommitment(requestRoot phase0.Root, signature phase0.BLSSignature) error
    // GetBlockHeader is for proposer duty
    GetBlockHeader() (ssz.Marshaler, spec.DataVersion, error)
}
```

`GetNewRequest()` is expected to return a new **valid** preconf request root each call, it returns an error if there is no new preconf request. 

`SubmitCommitment()` sends a signed preconf request root under the [commit boost domain](https://github.com/Commit-Boost/commit-boost-client/blob/c5a16eec53b7e6ce0ee5c18295565f1a0aa6e389/crates/common/src/constants.rs#L3) to the preconf protocol sidecar, which is responsible for communicating with relays and users.

(signing root and request root are different).

`GetBlockHeader()` allows the proposer to get a block header that satisfies the signed preconf requests the from relays. This function assumes to only return a valid block header as the preconf protocol sidecar can verify the block using Merkle proofs of transaction inclusion given the block header and the proof provided by the relays.

### Preconfirmation Duty Runner
This duty handles preconfirmation requests sent to validators. The preconf runner has access to the preconf sidecar (commit boost module).

```go
type PreconfRunner struct {
	BaseRunner *BaseRunner

	beacon         BeaconNode
	preconf        PreconfSidecar
	network        Network
	signer         types.BeaconSigner
	operatorSigner *types.OperatorSigner
	valCheck       qbft.ProposedValueCheckF

	requestRoot phase0.Root
}
```

The duty starts when a preconf request is received at the operators' local commit boost module. All operators are expected to receive the same requests from the commit boost module at the same time. 

```go
func (r *PreconfRunner) executeDuty(duty types.Duty) error {
	request, err := r.GetPreconfSidecar().GetNewRequest()
	if err != nil {
		return errors.Wrap(err, "failed to get preconf request")
	}

	r.requestRoot = request.Root

	msg, err := r.BaseRunner.signBeaconObject(r, r.BaseRunner.State.StartingDuty.(*types.ValidatorDuty), &request,
		duty.DutySlot(),
		types.DomainCommitBoost)
	if err != nil {
		return errors.Wrap(err, "failed signing attestation data")
	}
    ...
}
```

The request root is stored by the runner as the commit boost API request the root when submitting the signed request.

The signing root (request root + commit boost domain) is broadcasted to other operators.

When receiving other partial signatures, the runner submits the request root and the signature (over the signing root) to the commit boost API.

```go
func (r *PreconfRunner) ProcessPreConsensus(signedMsg *types.PartialSignatureMessages) error {
	quorum, roots, err := r.BaseRunner.basePreConsensusMsgProcessing(r, signedMsg)
	if err != nil {
		return errors.Wrap(err, "failed processing preconfirmation message")
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

	if err := r.GetPreconfSidecar().SubmitCommitment(r.requestRoot, specSig); err != nil {
		return errors.Wrap(err, "could not submit to commitment to sidecar")
	}

	r.GetState().Finished = true
	r.requestRoot = phase0.Root{}
	return nil
}
```

### Validator Commit Boost: Manager for Preconfirmation Duties
Since the current `Validator` duty manager is designed to have only one runner per duty, it is not aligned with the multiple duties requirement for preconf. Therefore, a new `ValidatorCommitBoost` duty manager is created to manage preconfirmation duty runners.

```go
type PreconfRunners map[phase0.Root]Runner

type ValidatorCommitBoost struct {
	PreconfRunners  PreconfRunners
	Network         Network
	CommitteeMember *types.CommitteeMember
	Share           *types.Share
	Signer          types.BeaconSigner
	OperatorSigner  *types.OperatorSigner
}
```

Unlike the `Validator` duty manager that routes duty to the an existing runner, the `ValidatorCommitBoost` duty manager handles preconf runner creation on receiving new preconf request.

```go
type PreconfDuty struct {
	RequestRoot phase0.Root `ssz-size:"32"`
	Slot        phase0.Slot `ssz-size:"8"`
}

func (v *ValidatorCommitBoost) StartDuty(duty types.PreconfDuty) error {
	_, exist := v.PreconfRunners[duty.RequestRoot]
	if exist {
		return errors.Errorf("duty runner for request %s already exists", duty.RequestRoot.String())
	}
	shareMap := make(map[phase0.ValidatorIndex]*types.Share)
	shareMap[v.Share.ValidatorIndex] = v.Share
	dutyRunner, err := NewPreconfRunner(v.BeaconNetwork, shareMap, v.Beacon, v.PreconfSidecar, v.Network, v.Signer, v.OperatorSigner)
	if err != nil {
		return errors.Wrap(err, "failed to create new preconf runner")
	}
	v.PreconfRunners[duty.RequestRoot] = dutyRunner
	return dutyRunner.StartNewDuty(duty, v.CommitteeMember.GetQuorum())
}

```

## Open Questions
- Should we enforce some rate limit on the preconf requests?
- How to enforce and check that all operators receive the same requests?
- Do we need to make sure/how to make sure they use the same relays?
- Can we get proofs for block headers from commit boost API? Scenario: a proposer leader proposer a bad block (e.g. because it uses a different relay)