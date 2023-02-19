| Author      | Title                 | Category | Status |
|-------------|-----------------------|----------|--------|
| Alon Muroch | Pre-Consensus Livness | Core     | open-for-discussion  |

**Summary**  
Some duties require pre-consensus (randao, selection proof, etc) before the consensus stage can start. Partial signatures are signed and broadcasted to reconstruct a valid signature for the pre-consensus step.

Once the pre-consensus step is done (has quorum of signatures) the consensus phase starts, with the pre-consensus result as input.

A livness issue is found when in T0 2f replicas are first to broadcast their partial pre-consensus signature, and receive it.  
The rest of the replicas do not receive the above messages.  
In T1 (still within the slot for the duty) another f+1 replicas broadcast their pre-consensus signatures.  

The first 2f replicas each received a quorum of pre-consensus message thus can start a consensus instance, the later f+1 replicas only received f+1 messages (from themselves). The second group of replicas (f+1) will **NEVER** start a consensus instance.

Since consensus instances must decide we find the system in a dead-lock since 2f replicas started a consensus instance, thus will **NEVER** broadcast a pre-consensus message until they decide, but they can't decided since f+1 replicas are still in the pre-consensus stage.  
The latter, f+1 group, will **NEVER** start a consensus instance since they will at most get f+1 pre-consensus messages.

The system is forever stuck.

**Specification**  
We add a pre-consensus justification field to the ConsensusData object (attached to all QBFT messages) so any replica which did not receive a quorum of pre-consensus messages can use the justification (verify it) and start a consensus instance.   
Solving the livness issue.

runner.go
```go
type BaseRunner struct {
	...

	// highestDecidedSlot holds the highest decided duty slot and gets updated after each decided is reached
	// this param is not part of the state as it should be set on struct init and updated during normal operations when decided
	highestDecidedSlot spec.Slot
}
```

partial_signature_message.go
```go
// PartialSignatureMessage is a msg for partial Beacon chain related signatures (like partial attestation, block, randao sigs)
type PartialSignatureMessage struct {
    PartialSignature []byte // The Beacon chain partial Signature for a duty
    SigningRoot      []byte // the root signed in PartialSignature
    Signer           OperatorID
}

type PartialSignatureMessages struct {
	Type     PartialSigMsgType
	Slot     phase0.Slot
	Messages []*PartialSignatureMessage
}

// SignedPartialSignatureMessage is an operator's signature over PartialSignatureMessage
type SignedPartialSignatureMessage struct {
    Message   PartialSignatureMessages
    Signature Signature
    Signer    OperatorID
}

```

consensus_data.go
```go
// ConsensusData holds all relevant duty and data Decided on by consensus
type ConsensusData struct {
    Duty                       Duty
    Version                    spec.DataVersion
    PreConsensusJustifications []*SignedPartialSignatureMessage `ssz-max:"13"`
    DataSSZ                    []byte                           `ssz-max:"1073807360"` // 2^30+2^16 (considering max block size 2^30)
}

func (cid *ConsensusData) Validate() error {
    ...
    
    role := cid.Duty.Type
    
    if role == BNRoleAggregator {
        ...
        if !cid.validateUniqueJustificationSigners() {
            return errors.New("invalid pre-consensus justification")
        }
    }
    
    if role == BNRoleProposer {
        ...
        if !cid.validateUniqueJustificationSigners() {
            return errors.New("invalid pre-consensus justification")
        }
    }
    
    if role == BNRoleSyncCommitteeContribution {
        ...
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

runner
```go
// canProcessPreConsensusJustification returns true if
// - there is no running instance
// - qbft message is controller.Height + 1
func (b *BaseRunner) canProcessPreConsensusJustification(msg *qbft.SignedMessage) bool {
    return b.QBFTController.CanStartInstance() == nil && b.QBFTController.Height+1 == msg.Message.Height
}

// validatePreConsensusJustification validates:
// 1) unique partial sig signers
// 2) quorum
// 3) slot valid
func (b *BaseRunner) validatePreConsensusJustificationForSlot(
    sigs []*types.SignedPartialSignatureMessage,
    slot phase0.Slot,
) error {
    signers := map[types.OperatorID]bool{}
    for _, sig := range sigs {
        if err := types.ValidateSignedPartialSignatureMessage(b.Share, sig, slot); err != nil {
            return err
        }
        if signers[sig.Signer] {
            return errors.New("non unique signers")
        }
        signers[sig.Signer] = true
    }
    return nil
}

// processPreConsensusJustification processes a pre-consensus justification
// highestDecidedDutySlot is the highest decided duty slot known
// is the qbft message carrying  the pre-consensus justification
func (b *BaseRunner) processPreConsensusJustification(runner Runner, highestDecidedDutySlot phase0.Slot, msg *qbft.SignedMessage) error {
    if !b.canProcessPreConsensusJustification(msg) {
        return nil
    }
    
    cd := &types.ConsensusData{}
    if err := cd.Decode(msg.Message.Data); err != nil {
        return err
    }
    
    err := b.validatePreConsensusJustificationForSlot(cd.PreConsensusJustifications, cd.Duty.Slot)
    if err != nil {
        return errors.Wrap(err, "invalid pre-consensus justification")
    }
    
    // only new pre-consensus justifications can be processed
    if cd.Duty.Slot <= highestDecidedDutySlot {
        return errors.New("invalid pre-consensus slot")
    }
    
    if !b.hasRunningDuty() {
        b.setupForNewDuty(&cd.Duty)
    }
    
    // add pre-consensus sigs to state container
    var r [][]byte
    for _, signedMsg := range cd.PreConsensusJustifications {
        quorum, roots, err := b.basePartialSigMsgProcessing(signedMsg, b.State.PreConsensusContainer)
        if err != nil {
            return errors.Wrap(err, "invalid partial sig processing")
        }
        
        if quorum {
            r = roots
            break
        }
    }
    if len(r) == 0 {
        return errors.New("invalid pre-consensus justification quorum")
    }
    
    return runner.decideOnRoots(r)
}

```