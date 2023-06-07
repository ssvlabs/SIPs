
| Author      | Title     | Category | Status |
|-------------|-----------|----------|--------|
| Alon Muroch | QBFT Sync | Core     | open-for-discussion  |

[Discussion] (https://github.com/bloxapp/SIPs/discussions/12)

**Summary**  
Describes when QBFT requires syncing with other nodes and what type of syncing.   
There are 2 types of syncing: 
1) Inter-instance syncing - syncing between instances, means syncing a nodes with all decided instances from some past instance height to the highest known decided instance 
2) Intra-instance syncing - syncing a node, in a specific instance which has yet to decide, with in-instance messages

**Why do we need syncing**  
Nodes might not receive all messages for various reasons (being offline, disconnected, etc) which can result in a node not receiving up-to-date decided messages (left behind on an old height) and/ or not receiving in-instance messages which might cause the entire committee to go into round changes.  
Round changes in QBFT are exponential to enable un-synced nodes to ultimately catch up. If a committee gets into high change rounds (timeouts can be hours and even days) it can take a significant amount of time for the committee to "return" back to executing duties.  
Introducing inter/ intra instance syncing can cut that time significantly and make the committee more lively.

_Example_: only 2 nodes are online starting instance at height H, after some time they are both on round R in which the round timeout is 2^R seconds.  
A third node goes back online, starting at height H' < H.   
Inter-instance syncing will enable the node to sync back to H, intra-instance syncing will enable the node to catchup quickly with messages leading to round R, compile a quorum and decided instance H.  

**What syncing doesn't solve**  
QBTF is a leader and round based protocol   requiring 2f+1 honest nodes to make progress.  
If t < f nodes are offline/ unstable the committee itself will see a degradation in performance as every time the leader is one of the offline nodes the committee will need to go into change round  

**Inter-instance syncing**  
A node receiving f+1 messages with a higher height than its local height will trigger sync highest decided in which the node fetches the highest decided (signed message and justification) from its peers.   
[UponFutureMsg code](https://github.com/bloxapp/ssv-spec/blob/qbft_sync/qbft/sync.go)  
[UponDecided code](https://github.com/bloxapp/ssv-spec/blob/qbft_sync/qbft/decided.go)

```go
// ProcessMsg processes a new msg, returns decided message or error
func (c *Controller) ProcessMsg(msg *SignedMessage) (*SignedMessage, error) {
    if err := c.baseMsgValidation(msg); err != nil {
        return nil, errors.Wrap(err, "invalid msg")
    }

    if isDecidedMsg(c.Share, msg) {
        return c.UponDecided(msg)
    } else if msg.Message.Height > c.Height {
        return c.UponFutureMsg(msg)
    } else {
        return c.UponExistingInstanceMsg(msg)
    }
}
```

All received decided messages from syncing will trigger, as usual, the upon decided function.  

_Important comment:_ F+1 messages are required to prevent malicous nodes forcing endless syncing from an honest node. In an F+1 (with max F malicious) there is at least 1 honest node sending a message with height > H.

**Intra-instance syncing**  
A node starting a new instance will sync get highest round change from peer, fetching the highest round change message (with the highest round) from peers.  
[Link to code](https://github.com/bloxapp/ssv-spec/blob/qbft_sync/qbft/instance.go#L65-L67)    

```go
func (i *Instance) Start(value []byte, height Height) {
    i.startOnce.Do(func() {
        // code before
		
        if err := i.config.GetNetwork().SyncHighestRoundChange(i.State.ID, i.State.Height); err != nil {
            // log
        }
    })
}
```

This sync only fetches the highest round change because for every round > FirstRound the following proposal requires a quorum of round change messages.  
Most times when a committee is "stuck" it's because it's waiting for such a quorum. Syncing directly the highest round change will enable the syncing node to quickly form a quorum and/ or speef up F+1 sync.

_**Continuous get highest round change**_  
Due to the importance of getting the most recent rc message we employ a continuous get highest round change in certain conditions.    
The above means that a get highest change round call will be triggered every X^(R-3) until the instance decides.

Params:  
Round Timeout(sec): X^R, R >= 0
Next get highest round change(sec): X^(R-3) ∀ R>6
Next get highest round frequency (within each round timeout) = X^R/X^(R-3)  = X^3

Example: X=2,R=10
Round timeout = 2^10 = 1024 seconds
Next get highest round change = 2^(10-3) = 128 seconds
Next get highest round frequency (within each round timeout) = 8

Example 2: X=3, R=10
Round timeout = 3^10 = 59049 seconds
Next get highest round change = 3^(10-3) = 2187 seconds
Next get highest round frequency (within each round timeout) = 27