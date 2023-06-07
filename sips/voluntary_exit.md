| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| Alon Muroch | Voluntary Exit | Core     | Draft  |


**Summary**
Describes how zero coordination voluntary exit works.  
<em>Important to note that a user holding the validator’s private key can sign a voluntary exit as well.</em>


**Specification**

**Contract**  
Adds a new function accessible only by the validator's owner, pushes a voluntray exit event. 
Can only be called once

**Networking**
A dedicated subnet for voluntary exits will be added to the network for all exit related communication between operators.
All operators subscribe to the network.

**Messages**
New SSV MsgType is introduced

| Name       | Type    | Value | Description                          |
|------------|---------|-------|--------------------------------------|
| SSVVoluntaryExit | MsgType | 3     | SSVMessage type for voluntary exit messages |

The SSVMessage Data field will encode the voluntary exit SignedMessage data structure. 

```go
// VoluntaryExit provides information about a voluntary exit.
type VoluntaryExit struct {
    Epoch          uint64
    ValidatorIndex uint64
}

type ExitMessage struct {
    ValidatorPubKey ValidatorPK `ssz-size:"48"`
    Message         *VoluntaryExit
}


type SignedExitMessage struct {
    // Signature using the ethereum address that registered the validator
    Signature [65]byte `ssz-size:"65"`
    Message   ExitMessage
}
```
**New Runner**  
A new runner for voluntary exits will be created.

Upon exit message being parsed from EL, each node taking part in the specific cluster will broadcast SignedExitMessage including a partial signature for the voluntary exit.
Upon a quorum of partial signatures, each node will reconstruct a valid signature and broadcast it to the beacon chain