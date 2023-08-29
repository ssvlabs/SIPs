| Author      | Title        | Category | Status |
|-------------|--------------|----------|--------|
| Alon Muroch & Matheus Franco | Fork Support | Core     | approved  |

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


## Current State

Currently, there's **no fork change logic**. However, there's **network and fork identification** through the `DomainType`. For instance, we have
```go
type DomainType [4]byte
var (
	GenesisMainnet = DomainType{0x0, 0x0, 0x0, 0x0}
	PrimusTestnet  = DomainType{0x0, 0x0, 0x1, 0x0}
	ShifuTestnet   = DomainType{0x0, 0x0, 0x2, 0x0}
	ShifuV2Testnet = DomainType{0x0, 0x0, 0x2, 0x1}
	V3Testnet      = DomainType{0x0, 0x0, 0x3, 0x1}
)
```
The _DomainType_ major importance is the fact that **it's mixed in the signature** such that messages on different domains will have different signatures.

The node takes one of the above _DomainType_ values and **holds it until termination**.

In the _DomainType_, the third byte corresponds to the **network** (Mainnet, Primus, etc.), and the fourth byte corresponds to the **fork or protocol version** for this network.

In the actual implementation, _DomainType_ appears in the following structures:
- `Share`
- `Config`
- `Controller`

Below, you can see how **DomainType** is linked with the main structures.

<p align="center">
<img src="./images/fork_support/domain_type_presence.png"  width="30%" height="10%">
</p>

### Use cases

- When an _Instance_ needs to verify a signature. For example
```go
signedMessage.Signature.VerifyByOperators(signedMessage, config.GetSignatureDomainType(), types.QBFTSignatureType, operators);
```


- Each _Runner_, by creating the message ID when it needs to create a message. For example
```go
msgToBroadcast := &types.SSVMessage{
		MsgType: types.SSVPartialSignatureMsgType,
		MsgID:   types.NewMsgID(r.GetShare().DomainType, r.GetShare().ValidatorPubKey, r.BaseRunner.BeaconRoleType),
		Data:    data,
	}
```

- When the _BaseRunner_ validates a partial signature. For instance
```go
func (b *BaseRunner) validatePartialSigMsgForSlot(...) {
	...
	signedMsg.GetSignature().VerifyByOperators(signedMsg, b.Share.DomainType, types.PartialSignatureType, b.Share.Committee)
	...
}
```

- Indirect usage: an _Instance_ includes its _State.ID_ in messages. The _State.ID_ is initialized as the controller's _Identifier_ which contains the _DomainType_. The controller's identifier is fixed. It's used as:
  - an argument on instance creation.
  - to broadcast decided messages.
  - in base validation, to verify if the message was sent to the correct controller.



## Proposed Changes

Along with the `DomainType`, we include a `NetworkID` type and a new `ForkData` structure.


```go
// NetworkID are intended to separate different SSV networks. A network can have many forks in it.
type NetworkID byte

const (
	MainnetNetworkID = NetworkID(0x0)
	PrimusNetworkID  = NetworkID(0x1)
	ShifuNetworkID   = NetworkID(0x2)
	JatoNetworkID    = NetworkID(0x3)
	JatoV2NetworkID    = NetworkID(0x4)
)

// DomainType is a unique identifier for signatures, 2 identical pieces of data signed with different domains will result in different sigs
type DomainType [4]byte

// DomainTypes represent specific forks for specific chains, messages are signed with the domain type making 2 messages from different domains incompatible
var (
	GenesisMainnet = DomainType{0x0, 0x0, byte(MainnetNetworkID), 0x0}
	PrimusTestnet  = DomainType{0x0, 0x0, byte(PrimusNetworkID), 0x0}
	ShifuTestnet   = DomainType{0x0, 0x0, byte(ShifuNetworkID), 0x0}
	ShifuV2Testnet = DomainType{0x0, 0x0, byte(ShifuNetworkID), 0x1}
	JatoTestnet    = DomainType{0x0, 0x0, byte(JatoNetworkID), 0x0}
	JatoV2Testnet  = DomainType{0x0, 0x0, byte(JatoV2NetworkID), 0x1}
)

// ForkData is a simple structure holding fork information for a specific chain (and its fork)
type ForkData struct {
	// Epoch in which the fork happened
	Epoch phase0.Epoch
	// Domain for the new fork
	Domain DomainType
}

func (domainType DomainType) GetNetworkID() NetworkID {
	return NetworkID(domainType[2])
}

func (networkID NetworkID) GetForksData() []*ForkData {
	switch networkID {
	case MainnetNetworkID:
		return mainnetForks()
	default:
		return []*ForkData{
			{
				Epoch:  0,
				Domain: DomainType{0x0, 0x0, byte(networkID), 0x0},
			},
		}
	}
}

// mainnetForks returns all forks for the mainnet chain
func mainnetForks() []*ForkData {
	return []*ForkData{
		{
			Epoch:  0,
			Domain: GenesisMainnet,
		},
	}
}

func (networkID NetworkID) DefaultFork() *ForkData {
	return networkID.GetForksData()[0]
}

// GetCurrentFork returns the ForkData with highest Epoch smaller or equal to "epoch"
func (networkID NetworkID) ForkAtEpoch(epoch phase0.Epoch) (*ForkData, error) {
	// Get list of forks
	forks := networkID.GetForksData()

	// If empty, raise error
	if len(forks) == 0 {
		return nil, errors.New("GetCurrentFork: fork list by GetForksData is empty.")
	}

	var current_fork *ForkData
	for _, fork := range forks {
		if fork.Epoch <= epoch {
			current_fork = fork
		}
	}
	return current_fork, nil
}

func (f ForkData) GetRoot() ([]byte, error) {
	byts, err := json.Marshal(f)
	if err != nil {
		return nil, errors.Wrap(err, "could not marshal ForkData")
	}
	ret := sha256.Sum256(byts)
	return ret[:], nil
}
```

### Structures access to types

- `Share`: It's more correct for the structure to hold the information on the network that it's in, instead of the network and fork number.
    Thus, _Share_ will contain a `NetworkID` field instead of a `DomainType` field.
- `Config`: Similarly, _Config_ should have a `NetworkID` instead of `DomainType`.
- `Controller`: It may drop its `DomainType` field since it has access to `config` and `Share`.

Regarding the indirect usage of Identifiers:
- `Instance` must keep its identifier to send properly formed messages.
- `Controller` may keep its identifier, but its _StartNewInstance_ method should receive as an argument the new identifier for the instance, as well as the new _DomainType_ to update its _config_. The controller, then, updates its identifier.

```go
func (c *Controller) StartNewInstance(height Height, value []byte, identifier []byte, domainType types.DomainType) error { // <-- identifier as argument
	if err := c.GetConfig().GetValueCheckF()(value); err != nil {
		return errors.Wrap(err, "value invalid")
	}

	// can't use <= because of height == 0 case
	if height < c.Height {
		return errors.New("attempting to start an instance with a past height")
	}

	// covers height == 0 case
	if c.StoredInstances.FindInstance(height) != nil {
		return errors.New("instance already running")
	}

    c.config.SetSignatureDomainType(domainType) // <-- updates its config's DomainType
    c.Identifier = identifier // <-- updates its identifier
	c.Height = height
	newInstance := c.addAndStoreNewInstance()
	newInstance.Start(value, height)

	c.forceStopAllInstanceExceptCurrent()

	return nil
}
```

The above change also requires an update in the `BaseRunner`. In the `decide` function, where `StartNewInstance` is called, an updated identifier needs to be passed as an argument as well as the new DomainType. For that, the _BaseRunner_ will have a new attribute `CurrentFork`, updated every time a new duty starts.

```go
type BaseRunner struct {
	State          *State
	Share          *types.Share
	QBFTController *qbft.Controller
	BeaconNetwork  types.BeaconNetwork
	BeaconRoleType types.BeaconRole
	highestDecidedSlot spec.Slot

	// Current fork associated to Share.DomainType
	CurrentFork *types.ForkData  // <-- new field
}

func (b *BaseRunner) baseSetupForNewDuty(duty *types.Duty) {

	// Update CurrentFork
	epoch := b.BeaconNetwork.EstimatedEpochAtSlot(duty.Slot)
	b.CurrentFork = b.Share.NetworkID.GetCurrentFork(epoch) // <-- update Current fork

	// start new state
	b.State = NewRunnerState(b.Share.Quorum, duty)
}


func (b *BaseRunner) decide(runner Runner, input *types.ConsensusData) error {
	byts, err := input.Encode()
	if err != nil {
		return errors.Wrap(err, "could not encode ConsensusData")
	}

	if err := runner.GetValCheckF()(byts); err != nil {
		return errors.Wrap(err, "input data invalid")
	}

	identifier := types.NewMsgID(b.CurrentFork.Domain, b.Share.ValidatorPubKey[:], b.BeaconRoleType) // <-- computes new identifier

	if err := runner.GetBaseRunner().QBFTController.StartNewInstance(
		qbft.Height(input.Duty.Slot),
		byts,
		identifier[:], // <-- passes identifier as argument
        b.CurrentFork.Domain, // <-- passes new domain as argument
	); err != nil {
		return errors.Wrap(err, "could not start new QBFT instance")
	}
	newInstance := runner.GetBaseRunner().QBFTController.InstanceForHeight(runner.GetBaseRunner().QBFTController.Height)
	if newInstance == nil {
		return errors.New("could not find newly created QBFT instance")
	}

	runner.GetBaseRunner().State.RunningInstance = newInstance
	return nil
}
```
