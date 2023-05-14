| Author      | Title        | Category | Status |
|-------------|--------------|----------|--------|
| Alon Muroch | Fork Support | Core     | open-for-discussion  |

**Summary**  
Describes how to support forks in the SSV network.
A fork is a non backwards compatible change to the protocol, i.e., change message structs/ types/ functionality.

Forks are a bit tricky in SSV as there are multiple QBFT instances running with no way of knowing when each ends.  
For a fork to work it needs to:
1) Clear and deterministic trigger
2) [Consider weak subjectivity](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/weak-subjectivity/) - If an offline node comes to life and starts syncing messages it needs to be able to validate them, pre- and post-fork.

A fork will be triggered by a beacon epoch number as its deterministic and easy to compute independently. If epoch E is the trigger epoch then pre-fork are all messages refering epoch < E and post-fork are all messages referring epoch >= E.

[Suggested changes to SSV-Spec](https://github.com/bloxapp/ssv-spec/compare/main...alonmuroch:ssv-spec:ssv-fork-updated)

**Difficulty tracking forks**  
Every SSV node runs multiple validators for which 5 different duty runners are creates (attestation, aggregation, proposal, sync committee and sync committee contribution). Each of the runners have an independent QBFT controller running sequential instances (one can start after the previous has decided).

Some duties have a pre-consensus phase (for example proposer needs to sign randao pre-consensus), all duties have a consensus and post consensus phases.
Only when the consensus phase starts the QBFT controller must wait until it decides, which can takes 500ms or up to days if there is no quorum of peers.

**Triggering Fork**  
A fork is triggered by a specific epoch E.
Different forks might require different message processing, parsing and more which should be detailed in the spec.

**Spec Changes** 

BaseRunner changes
```go
// setupForNewDuty is sets the runner for a new duty
func (b *BaseRunner) baseSetupForNewDuty(duty *types.Duty) error {
    // handle fork change
    currentFork, err := b.forkBasedOnLatestDecided()
    if err != nil {
        return errors.Wrap(err, "could not calculate current fork")
    }
    b.Share.DomainType = currentFork
    b.QBFTController.SetDomainType(currentFork)
    
    // start new state
    b.State = NewRunnerState(b.Share.Quorum, duty)
    
    return nil
}

// forkBasedOnLatestDecided will return domain type based on b.Height instance that was previously decided
func (b *BaseRunner) forkBasedOnLatestDecided() (types.DomainType, error) {
    inst := b.QBFTController.InstanceForHeight(b.QBFTController.Height)
    if inst == nil {
        return b.Share.DomainType, nil
    }
    
    _, decidedValue := inst.IsDecided()
    cd := &types.ConsensusData{}
    if err := cd.Decode(decidedValue); err != nil {
        return b.Share.DomainType, errors.Wrap(err, "could not decoded consensus data")
    }
    
    currentForkDigest := b.Share.DomainType
    for _, forkData := range currentForkDigest.GetForksData() {
        if b.BeaconNetwork.EstimatedEpochAtSlot(cd.Duty.Slot) >= forkData.Epoch {
            currentForkDigest = forkData.Domain
        }
    }
    return currentForkDigest, nil
}
```

type changes
```go
// DomainType is a unique identifier for signatures, 2 identical pieces of data signed with different domains will result in different sigs
type DomainType [4]byte

const (
    GenesisChain = 0x0
    PrimusChain  = 0x1
    ShifuChain   = 0x2
    JatoChain    = 0x3
)

var (
    GenesisMainnet = DomainType{0x0, 0x0, GenesisChain, 0x0}
    PrimusTestnet  = DomainType{0x0, 0x0, PrimusChain, 0x0}
    ShifuTestnet   = DomainType{0x0, 0x0, ShifuChain, 0x0}
    ShifuV2Testnet = DomainType{0x0, 0x0, ShifuChain, 0x1}
    JatoTestnet    = DomainType{0x0, 0x0, JatoChain, 0x1}
)

type ForkData struct {
    Epoch  phase0.Epoch
    Domain DomainType
}

func (domainType DomainType) GetChain() byte {
    return domainType[2]
}

func (domainType DomainType) GetForksData() []*ForkData {
    switch domainType.GetChain() {
    case GenesisChain:
        return genesisForks()
    default:
        return []*ForkData{}
    }
}

func genesisForks() []*ForkData {
    return []*ForkData{
        {
            Epoch:  0,
            Domain: GenesisMainnet,
        },
    }
}

```

**Syncing considerations**  
Sycing messages (past or future instances) will require special considerations.
When syncing from a peer, it will provide the requesting node with the fork for the specific message sent.
For example: if my node asks a peer for the highest decided, the peer will return it + indicate which fork the message uses.