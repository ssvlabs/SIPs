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

[Suggested changes to SSV-Spec](https://github.com/bloxapp/ssv-spec/compare/main...alonmuroch:ssv-spec:experiments/fork-support)

**Difficulty tracking forks**  
Every SSV node runs multiple validators for which 5 different duty runners are creates (attestation, aggregation, proposal, sync committee and sync committee contribution). Each of the runners have an independent QBFT controller running sequential instances (one can start after the previous has decided).

Some duties have a pre-consensus phase (for example proposer needs to sign randao pre-consensus), all duties have a consensus and post consensus phases.
Only when the consensus phase starts the QBFT controller must wait until it decides, which can takes 500ms or up to days if there is no quorum of peers.

**Triggering Fork**  
A fork is triggered by a specific epoch E detailed in the ForkData(see below).
Different forks might require different message processing, parsing and more which should be detailed in the spec.

We check fork digest version whenever baseStartNewDuty is called and passes the canStartNewDuty check.

**Spec Changes** 

BaseRunner changes
```go
type BaseRunner struct {
    State           *State
    Share           *types.Share // domain type is removed from Share
    ForkDigest      types.ForkDigest
    QBFTController  *qbft.Controller
    BeaconNetwork   types.BeaconNetwork
    SSVNetworkChain types.SSVNetworkChain
    BeaconRoleType  types.BeaconRole
}

// baseStartNewDuty is a base func that all runner implementation can call to start a duty
func (b *BaseRunner) baseStartNewDuty(runner Runner, duty *types.Duty) error {
	if err := b.canStartNewDuty(); err != nil {
		return err
	}

	// Based on previous decided instance we calculate the fork digest to be used on the next instance
	currentForkDigest, err := b.forkBasedOnLatestDecided()
	if err != nil {
		return errors.Wrap(err, "could not calculate current fork digest")
	}
	b.ForkDigest = currentForkDigest

	b.State = NewRunnerState(b.Share.Quorum, duty)
	return runner.executeDuty(duty)
}

// forkBasedOnLatestDecided will return fork digest based on b.Height instance that was previously decided
func (b *BaseRunner) forkBasedOnLatestDecided() (types.ForkDigest, error) {
    inst := b.QBFTController.InstanceForHeight(b.QBFTController.Height)
    if inst == nil {
        return b.SSVNetworkChain.DefaultForkDigest(), nil
    }
    
    _, decidedValue := inst.IsDecided()
    cd := &types.ConsensusData{}
    if err := cd.Decode(decidedValue); err != nil {
        return types.ForkDigest{}, errors.Wrap(err, "could not decoded consensus data")
    }
    
    currentForkDigest := b.SSVNetworkChain.DefaultForkDigest()
    for _, forkData := range b.SSVNetworkChain.GetForksData() {
        if b.BeaconNetwork.EstimatedEpochAtSlot(cd.Duty.Slot) >= forkData.Epoch {
            currentForkDigest = forkData.CalculateForkDigest()
        }
    }
    return currentForkDigest, nil
}
```

type changes
```go
type SSVNetworkChain []byte

var (
    MainnetSSVNetworkChain      SSVNetworkChain = []byte("mainnet")
    ShifuTestnetSSVNetworkChain SSVNetworkChain = []byte("shifu_testnet")
)

func (chain SSVNetworkChain) GetForksData() []ForkData {
    return []ForkData{
        {
            Epoch:             0,
            GenesisIdentifier: chain,
        },
    }
}

func (chain SSVNetworkChain) DefaultForkDigest() ForkDigest {
    return chain.GetForksData()[0].CalculateForkDigest()
}

type ForkData struct {
    // Epoch for which the fork is triggered (for all messages >= Epoch)
    Epoch spec.Epoch
    // GenesisIdentifier is a unique constant identifier per chain
    GenesisIdentifier []byte
}

func (dd ForkData) GetRoot() ([]byte, error) {
    byts, err := json.Marshal(dd)
    if err != nil {
        return nil, errors.Wrap(err, "could not marshal ForkData")
    }
    ret := sha256.Sum256(byts)
    return ret[:], nil
}

// CalculateForkDigest returns the fork digest for the fork data
func (dd ForkData) CalculateForkDigest() ForkDigest {
    r, err := dd.GetRoot()
    if err != nil {
        panic(err.Error())
    }
    ret := ForkDigest{}
    copy(ret[:], r[:4])
    
    return ret
}

// ForkDigest is a 4 byte identifier for a specific fork, calculated from ForkData
type ForkDigest [4]byte

```

**Syncing considerations**  
Sycing messages (past or future instances) will require special considerations.
When syncing from a peer, it will provide the requesting node with the fork for the specific message sent.
For example: if my node asks a peer for a the highest decided, the peer will return it + indicate which fork the message uses.