| Author      | Title                 | Category | Status |
|-------------|-----------------------|----------|--------|
| Alon Muroch | Pre-Consensus Livness | Core     | open-for-discussion  |

**Summary**  
Currently duties requiring pre-consensus (randao, selection proof, etc) have a step before consensus where partial signatures are signed and broadcasted to reconstruct a valid signature for the pre-consensus step.

Once the pre-consensus step is done (has quorum of signatures) the consensus phase starts, with the pre-consensus result as input.

A livness issue is detected where in T0 2f replicas are first to broadcast their partial pre-consensus signature, and receive it.  
The rest of the replicas do not receive the above messages.  
In T1 (still within the slot for the duty) another f+1 replicas broadcast their pre-consensus signatures.  

The first 2f replicas each received a quorum of pre-consensus message thus can start a consensus instance, the later f+1 replicas only received f+1 messages (from themselves). The second group of replicas (f+1) will not start a consensus instance.

Since consensus instances must decide we find the system in a dead-lock since 2f replicas started a consensus instance, thus will not broadcast a pre-consensus message until they decide, but they can't decided since f+1 replicas are still in the pre-consensus stage.  
The latter, f+1 group, will never start a consensus instance since they will at most get f+1 pre-consensus messages.

The system is forever stuck.

**Specification**  
We add a pre-consensus justification field to the ConsensusData object (attached to all QBFT messages) so any replica which did not receive a quorum of pre-consensus message can use the justification (verify it) and start a consensus instance.   
Solving the livness issue.

consensus_data.go
```go
// ConsensusData holds all relevant duty and data Decided on by consensus
type ConsensusData struct {
    Duty                      *Duty
    PreConsensusJustification []*SignedPartialSignatureMessage
    AttestationData           *phase0.AttestationData
    BlockData                 *bellatrix.BeaconBlock
    BlindedBlockData          *bellatrix2.BlindedBeaconBlock
    AggregateAndProof         *phase0.AggregateAndProof
    SyncCommitteeBlockRoot    phase0.Root
    // SyncCommitteeContribution map holds as key the selection proof for the contribution
    SyncCommitteeContribution ContributionsMap
}

func (cid *ConsensusData) Validate() error {
    if cid.Duty == nil {
        return errors.New("duty is nil")
    }
    
    role := cid.Duty.Type
    
    if role == BNRoleAttester && cid.AttestationData == nil {
        return errors.New("attestation data is nil")
    }
    
    if role == BNRoleAggregator {
        if cid.AggregateAndProof == nil {
            return errors.New("aggregate and proof data is nil")
        }
        if !cid.validateUniqueJustificationSigners() {
            return errors.New("invalid pre-consensus justification")
        }
    }
    
    if role == BNRoleProposer {
        if cid.BlockData == nil && cid.BlindedBlockData == nil {
            return errors.New("block data is nil")
        }
    
        if cid.BlockData != nil && cid.BlindedBlockData != nil {
            return errors.New("block and blinded block data are both != nil")
        }
        
        if !cid.validateUniqueJustificationSigners() {
            return errors.New("invalid pre-consensus justification")
        }
    }
    
    if role == BNRoleSyncCommitteeContribution {
        if len(cid.SyncCommitteeContribution) == 0 {
            return errors.New("sync committee contribution data is nil")
        }
        if !cid.validateUniqueJustificationSigners() {
            return errors.New("invalid pre-consensus justification")
        }
    }
    
    return nil
}
    
func (cid *ConsensusData) validateUniqueJustificationSigners() bool {
    if cid.PreConsensusJustification == nil {
        return false
    }
    
    // check unique signers
    signed := make(map[OperatorID]bool)
    for _, msg := range cid.PreConsensusJustification {
        if msg.Validate() != nil {
			return false
        }
    
        if signed[msg.Signer] {
            return false
        }
        if msg.Signer == 0 {
            return false
        }
        signed[msg.Signer] = true
    }
    
    return true
}
```

runner.go
```go
func (b *BaseRunner) processPreConsensusJustification(msg *qbft.SignedMessage) error {
	// running instance already has pre-consensus justification
	if b.hasRunningDuty() {
		return nil
	}

	// TODO - should it be applied for this only?
	if b.QBFTController.Height+1 != msg.Message.Height {
		return nil
	}

	cd := types.ConsensusData{}
	if err := cd.Decode(msg.Message.Data); err != nil {
		return errors.Wrap(err, "could not decode consensus data")
	}

	if err := cd.Validate(); err != nil {
		return errors.Wrap(err, "invalid consensus data")
	}

	if !b.Share.HasQuorum(len(cd.PreConsensusJustification)) {
		return errors.New("no pre-consensus quorum")
	}

	// TODO - should we check if it's an actual pre-consensus msg?

	// validate signatures, a pre-cursor for a valid justification
	for _, preConsMsg := range cd.PreConsensusJustification {
		if err := b.validateSignedPartialSigMsg(preConsMsg, cd.Duty.Slot); err != nil {
			return err
		}
	}

	b.State = NewRunnerState(b.Share.Quorum, cd.Duty)
	for _, preConsMsg := range cd.PreConsensusJustification {
		for _, m := range preConsMsg.Message.Messages {
			b.State.PreConsensusContainer.AddSignature(m)
		}
	}

	return nil
}

// baseConsensusMsgProcessing is a base func that all runner implementation can call for processing a consensus msg
func (b *BaseRunner) baseConsensusMsgProcessing(runner Runner, msg *qbft.SignedMessage) (decided bool, decidedValue *types.ConsensusData, err error) {
    if err := b.processPreConsensusJustification(msg); err != nil {
        return false, nil, errors.Wrap(err, "invalid pre-consensus justification")
    }
	
    ...
}
```