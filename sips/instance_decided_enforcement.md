| Author      | Title                        | Category | Status |
|-------------|------------------------------|----------|--------|
| Alon Muroch | Instance Decided Enforcement | Core     | open-for-discussion  |

**Summary**

No longer require a QBFT instance to decide in order to start the next one.
If an instance is stalled, it will be terminated after a certain round or when a new instance starts.
The instance's height will be the duty's slot number. This will allow for easier syncing.

**Rationale**

In our current implementation, a QBFT instance can only start if the previous one has decided.
This is done to ensure that all nodes are synchronized and that the instance height is predictable.
This is not required since we can use slots to sync decided instances and the instance's height can be the duty's slot number.

The change will allow for more efficient constant timeouts without risking the liveness of the protocol. Currently, there is a liveness risk with constant timeouts.
F + 1 nodes that stall behind the committee's rounds can delay consensus indefinitely and prevent the start of new instances.
Since after 2 epochs, the duty will expire and will not be able to gain rewards, we can terminate the instance after a certain round.
If a new instance starts, the previous one will be terminated. This is to reduce slashing risks and to save on resources. 

**Spec Changes** 

**Timeout**
~~~go
var (
	quickTimeoutThreshold = Round(8)
	quickTimeout          = 2 * time.Second
	slowTimeout           = 2 * time.Minute 
	// CutoffRound which round the instance should stop its timer and progress no further 
	CutoffRound = 15 // stop processing instances after 8*2+120*6 = 14.2 min (~ 2 epochs)
)

~~~

**QBFT Controller**
- Remove function CanStartInstance
- Remove function canStartInstanceForValue
- Remove function bumpHeight
- Remove future message processing - no need as each instance is independent. If a future decided message is received it will trigger decided sync
```go
// StartNewInstance will start a new QBFT instance, if can't will return error
func (c *Controller) StartNewInstance(height Height, value []byte) error {
	if err := c.GetConfig().GetValueCheckF()(value); err != nil {
		return errors.Wrap(err, "value invalid")
	}
	
	// cab't use <= because of height == 0 case
	if height < c.Height {
		return errors.New("invalid instance height")
	}
	
	// covers height == 0 case
	if c.StoredInstances.FindInstance(height) != nil {
		return errors.New("instance already running")
	}
	
	c.Height = height

	newInstance := c.addAndStoreNewInstance()
	newInstance.Start(value, height)

	
	c.forceStopAllInstanceExceptCurrent()

	return nil
}

func (c *Controller) forceStopAllInstanceExceptCurrent() {
    for _, i := range c.StoredInstances {
        if i.State.Height != c.Height {
            i.ForceStop()
        }
    }
}

```

**QBFT Instance**
```go
type Instance struct {
	// ...
    // forceStop will force stop the instance if set to true
    forceStop  bool
}

func (i *Instance) ForceStop() {
    i.forceStop = true
}


// CanProcessMessages will return true if instance can process messages
func (i *Instance) CanProcessMessages() bool {
	return i.State.Round < CutoffRound && !i.forceStop
}
```
```go
// ProcessMsg processes a new QBFT msg, returns non nil error on msg processing error
func (i *Instance) ProcessMsg(msg *SignedMessage) (decided bool, decidedValue []byte, aggregatedCommit *SignedMessage, err error) {
    if i.CanProcessMessages() {
        return false, nil, nil, errors.New("instance stopped processing messages")
    }
	
	// ...
}
```
```go
func (i *Instance) UponRoundTimeout() error {
	if i.CanProcessMessages() {
		return errors.New("instance stopped processing timeouts")
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
- Start new instance with the duty's slot
```go
// ProcessMsg processes a new QBFT msg, returns non nil error on msg processing error
if err := runner.GetBaseRunner().QBFTController.StartNewInstance(
    qbft.Height(input.Duty.Slot),
    byts,
); err != nil {
    return errors.Wrap(err, "could not start new QBFT instance")
}
```

**Syncing**  
- Light nodes do not need to sync highest decided at all
- Syncing for decided messages is done by epoch range

**Non Committee Message Processing**  
- Light nodes do not need to process and store decided messages (to serve them for get highest decided requests)