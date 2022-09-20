| Author      | Title         | Category | Status |
|-------------|---------------|----------|--------|
| Alon Muroch | Full SSV Node | Core     | draft  |

## Overview  
Current QBFT controller does not force local storage of all past decided instances, rather just the highest decided is promnsied to be saved.  
This SIP introduces a "full-node" concept where a node will maintain the full history for every QBFT controller, syncing from other nodes if data is unavailable. 

We introduce a new syncing route and message handling.

[Experimental code branch](https://github.com/bloxapp/ssv-spec/tree/full_node/qbft)

## Changed Interfaces 
```go
// Storage is a collection of persistent storage functions
type Storage interface {
	// SaveHighestDecided saves (and potentially overrides) the highest Decided for a specific instance
	SaveHighestDecided(signedMsg *SignedMessage) error
	// GetHighestDecided returns highest decided if found, nil if didn't
	GetHighestDecided(identifier []byte) (*SignedMessage, error)
    // SaveDecided saves the decided message (for the identifier and height)
    SaveDecided(msg *SignedMessage) error
    // GetDecided returns a non nil message if found, nil otherwise
    GetDecided(identifier []byte, height Height) (*SignedMessage, error)
}

type Sync interface {
    // SyncHighestDecided tries to fetch the highest decided from peers (not blocking)
    SyncHighestDecided(identifier []byte) error
    // SyncHighestRoundChange tries to fetch for each committee member the highest round change broadcasted for the specific height from peers (not blocking)
    SyncHighestRoundChange(identifier []byte, height Height) error
	// SyncDecided will start an async syncing of decided message from to specified heights
	SyncDecided(identifier []byte, from, to Height) error
}
```
## Changes to QBFT Controller

### Remove HistoricalInstanceCapacity
QBFT controller will maintain just the current running instance, other instances will be fetched from storage

### Controller
```go
type Controller struct {
    Identifier []byte
    Height     Height // incremental Height for InstanceContainer
    // StoredInstances stores the last HistoricalInstanceCapacity in an array for message processing purposes.
    RunningInstance *Instance
    // FutureMsgsContainer holds all msgs from a higher height
    FutureMsgsContainer map[types.OperatorID]Height // maps msg signer to height of higher height received msgs
    Domain              types.DomainType
    Share               *types.Share
    config              IConfig
}

// ProcessMsg processes a new msg, returns decided message or error
func (c *Controller) ProcessMsg(msg *SignedMessage) (*SignedMessage, error) {
	if err := c.baseMsgValidation(msg); err != nil {
		return nil, errors.Wrap(err, "invalid msg")
	}

	/**
	Main controller processing flow
	_______________________________
	All decided msgs are processed the same, out of instance
	All valid future msgs are saved in a container and can trigger highest decided sync
	All valid past msgs are processed separately
	All other msgs (not future or decided) are processed normally by an existing instance (if found)
	*/
	if isDecidedMsg(c.Share, msg) {
		return c.UponDecided(msg)
	} else if msg.Message.Height == c.Height {
		return c.UponExistingInstanceMsg(msg)
	} else if msg.Message.Height > c.Height {
		return c.UponFutureMsg(msg)
	} else {
		return nil, c.UponPastInstanceMsg(msg)
	}
}

func (c *Controller) UponPastInstanceMsg(msg *SignedMessage) error {
    if err := c.baseMsgValidation(msg); err != nil {
        return errors.Wrap(err, "invalid msg")
    }
    // accept only past commit msgs
    if msg.Message.MsgType != CommitMsgType {
        return errors.New("accepts only commit msgs for past instances")
    }

    existingDecided, err := c.GetConfig().GetStorage().GetDecided(c.Identifier, msg.Message.Height)
    if err != nil {
        return errors.Wrap(err, "could not fetch past decided")
    }
    if existingDecided == nil {
        return errors.New("expected existing past decided")
    }

    if err := existingDecided.Aggregate(msg); err != nil {
        return errors.Wrap(err, "could not aggregate commit msg")
    }
    return c.GetConfig().GetStorage().SaveDecided(existingDecided)
}
```

### UponDecided
```go
// UponDecided returns decided msg if decided, nil otherwise
func (c *Controller) UponDecided(msg *SignedMessage) (*SignedMessage, error) {
    // decided msgs for past (already decided) instances are processed differently
    if msg.Message.Height < c.Height {
        return nil, c.UponPastInstanceMsg(msg)
    }

    // ...

    if isFutureDecided {
        // ...
		
        if err := c.GetConfig().GetNetwork().SyncDecided(c.Identifier, c.Height, msg.Message.Height); err != nil {
            fmt.Printf(err.Error())
        }
        // bump height
        c.Height = msg.Message.Height 
		
        // ...
    }
}

func (c *Controller) UponPastDecided(msg *SignedMessage) error {
    if err := validateDecided(
        c.config,
        msg,
        c.Share,
    ); err != nil {
        return errors.Wrap(err, "invalid decided msg")
    }

    existingDecided, err := c.GetConfig().GetStorage().GetDecided(c.Identifier, msg.Message.Height)
    if err != nil {
        return errors.Wrap(err, "could not fetch past decided")
    }
    if existingDecided == nil {
        return c.GetConfig().GetStorage().SaveDecided(msg)
    } else {
        if !bytes.Equal(existingDecided.Message.Data, msg.Message.Data) {
            return errors.New("new decided msg != stored decided")
        }

        if len(msg.Signers) > len(existingDecided.Signers) {
            return c.GetConfig().GetStorage().SaveDecided(msg)
        }
    }
    return nil
}
```
