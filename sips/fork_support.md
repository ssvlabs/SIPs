| Author      | Title        | Category | Status |
|-------------|--------------|----------|--------|
| Alon Muroch | Fork Support | Core     | approved  |

**Summary**  
Describes how to support forks in the SSV network.
A fork is a non backwards compatible change to the protocol, i.e., change message structs/ types/ functionality.

Forks are a bit tricky in SSV as there are multiple QBFT instances running with no way of knowing when each ends.  
For a forks to work they need a clear trigger.

**Triggering**  
A fork will be triggered by a beacon epoch number as its deterministic and easy to compute independently. If epoch E is the trigger epoch then pre-fork are all messages refering epoch < E and post-fork are all messages referring epoch >= E.

**Transition**  
Once Epoch E starts, the runner and QBFT controller change domain type from T to T' (the new fork domain). 

Client implementation should calculate when a new duty starts to decide if it starts with T or T', transition during duty execution is impossible. 

**History Sync**  
Sycing messages from a peer stays unchanged as messages have the identifier in them with domain type 

[Suggested changes to SSV-Spec](https://github.com/bloxapp/ssv-spec/compare/main...alonmuroch:ssv-spec:ssv-fork-updated)

**Spec Changes**

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

