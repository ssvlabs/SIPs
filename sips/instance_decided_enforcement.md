| Author      | Title                        | Category | Status |
|-------------|------------------------------|----------|--------|
| Alon Muroch | Instance Decided Enforcement | Core     | open-for-discussion  |

**Summary** 
Describes the remove of requiring a QBFT instance to decided to start the next one. 

**Rational**  
Currently QBFT instances have an incremental index for their height, starting with 0.
For a new instance to start, the QBFT controller checks that the previous one has decided.
Originally this was made so instance heights are predictable, synchronized and easy to sync by every node in the network.

The main reasons to drop these requirements are:
1) Instance Height can be the duty's epoch number as every duty, at most, there is 1 duty per type of duty per validator.
2) Syncing historical decided instances can be made easy with the use of epochs, e.g. fetch decided instances for epochs 100-150.
3) Stalled/ non decided duties have a life expectancy of, at most, 2 epochs, since past that even if executed will not gain rewards for validator (per beacon roles)
4) There is a difficult balance between efficient constant time timeouts and liveness. There are some complex solutions but since duties have an effective expiration time and are "non-dependent" we can terminate at a certain round.

**Spec Changes** 

**QBFT Controller**
- Remove function CanStartInstance
- Remove function canStartInstanceForValue
- Remove function bumpHeight
```go
// StartNewInstance will start a new QBFT instance, if can't will return error
func (c *Controller) StartNewInstance(height Height, value []byte) error {
	if err := c.GetConfig().GetValueCheckF()(value); err != nil {
		return errors.Wrap(err, "value invalid")
	}

	// only if current height's instance exists (and decided since passed can start instance) bump
	if c.StoredInstances.FindInstance(height) != nil {
		return errors.New("instance already running")
	}

	newInstance := c.addAndStoreNewInstance()
	newInstance.Start(value, height)

	c.Height = height

	return nil
}
```

**QBFT Instance**
```go
// ProcessMsg processes a new QBFT msg, returns non nil error on msg processing error
func (i *Instance) ProcessMsg(msg *SignedMessage) (decided bool, decidedValue []byte, aggregatedCommit *SignedMessage, err error) {
    if i.State.Round == CutoffRound {
        return false, nil, nil, errors.New("round > cutoff round")
    }
	
	// ...
}
```
```go
// CutoffRound which round the instance should stop its timer and progress no further
const CutoffRound = 20

func (i *Instance) UponRoundTimeout() error {
	if i.State.Round == CutoffRound {
		return errors.New("round > cutoff round")
	}

	newRound := i.State.Round + 1
	defer func() {
		i.State.Round = newRound
		i.State.ProposalAcceptedForCurrentRound = nil
		i.config.GetTimer().TimeoutForRound(i.State.Round)
	}()

	roundChange, err := CreateRoundChange(i.State, i.config, newRound, i.StartValue)
	if err != nil {
		return errors.Wrap(err, "could not generate round change msg")
	}

	if err := i.Broadcast(roundChange); err != nil {
		return errors.Wrap(err, "failed to broadcast round change message")
	}

	return nil
}
```

**Runner**
- Remove function canStartNewDuty
- Start new instance on decide(runner Runner, input *types.ConsensusData) error will start the instance with the epoch calculated from the duty slot
```go
// ProcessMsg processes a new QBFT msg, returns non nil error on msg processing error
if err := runner.GetBaseRunner().QBFTController.StartNewInstance(
    qbft.Height(b.BeaconNetwork.EstimatedEpochAtSlot(input.Duty.Slot)),
    byts,
); err != nil {
    return errors.Wrap(err, "could not start new QBFT instance")
}
```