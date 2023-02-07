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

To avoid abuses by f byzantine nodes we wait for at least f+1 pre-consensus justification messages signed be unique replicas.

consensus_data.go
```go
// ConsensusData holds all relevant duty and data Decided on by consensus
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
// shouldProcessPreConsensusJustification returns true if there is no running instance and qbft message is controller.Height + 1
func (b *BaseRunner) shouldProcessPreConsensusJustification(msg *qbft.SignedMessage) bool {
    return b.QBFTController.CanStartInstance() == nil && b.QBFTController.Height+1 == msg.Message.Height
}

// validatePreConsensusJustification validates:
// 1) the qbft msg
// 2) the justifications
// 3) can add to container (unique signer)
// and returns the decoded consensus data
func (b *BaseRunner) validatePreConsensusJustification(runner Runner, msg *qbft.SignedMessage) (*types.ConsensusData, error) {

}

func (b *BaseRunner) hasPartialQuorumForPreConsensusJustification() bool {
    // check pre consensus roots equal
    // check duties are equal

    return b.Share.HasPartialQuorum(len(b.preConsensusJustificationContainer))
}

func (b *BaseRunner) processPreConsensusJustification(runner Runner, msg *qbft.SignedMessage) error {
    if !b.shouldProcessPreConsensusJustification(msg) {
        return nil
    }
    
    cd, err := b.validatePreConsensusJustification(runner, msg)
    if err != nil {
        return errors.Wrap(err, "invalid pre-consensus justification")
    }

    // add to f+1 container
    b.preConsensusJustificationContainer[msg.Signers[0]] = msg
    
    if !b.hasPartialQuorumForPreConsensusJustification() {
        return nil
    }

    if !b.hasRunningDuty() {
        b.setupForNewDuty(cd.Duty)
    }
    
    // add pre-consensus sigs to state container
    var r [][]byte
    for _, signedMsg := range cd.PreConsensusJustification {
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