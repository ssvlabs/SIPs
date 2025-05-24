| Author      | Title                   | Category | Status              | Date       |
|-------------|-------------------------|----------|---------------------|------------|
| Alan Li     | Remove decided message  | Core     | open-for-discussion | 2025-05-23 |

## Summary  
In QBFT consensus, decided message is a protocol message that consists of at least a quorum of valid commit messages. Operators can instantly terminate consensus if a valid decided message is received. However, it is an optional message to help nodes to terminate consensus. Operators are able to terminate consensus as long as a quorum of valid commit messages is received. This SIP proposes to remove the decided message from the consensus protocol.

## Motivation
Since it is not a required message, removing the decided message will reduce the processing overhead to an operator. Currently, decided messages accounted for 6.3% of the bandwidth and 3.7% of the messages in the network. Removing the decided message will reduce the bandwidth and the messages by 6.3% and 3.7% respectively.

<p align="center">
<img src="./images/remove_decided_msg/curr_bandwidth.png"  width="45%" height="10%">
<img src="./images/remove_decided_msg/curr_msgs.png"  width="45%" height="10%">
</p>

## Specification
There will be two stages of changes to remove the decided message.

### Stage 1: Stop broadcasting decided messages
Operators will stop broadcasting decided messages when the consensus is terminated.

```go
func (c *Controller) UponExistingInstanceMsg(msg *ProcessingMessage) (*types.SignedSSVMessage, error) {

	inst := c.InstanceForHeight(msg.QBFTMessage.Height)
	if inst == nil {
		return nil, errors.New("instance not found")
	}

	prevDecided, _ := inst.IsDecided()

	// if previously decided, we don't process more messages
	if prevDecided {
		return nil, errors.New("not processing consensus message since instance is already decided")
	}

	decided, _, decidedMsg, err := inst.ProcessMsg(msg)
	if err != nil {
		return nil, errors.Wrap(err, "could not process msg")
	}

	// save the highest Decided
	if !decided {
		return nil, nil
	}

    // ---------- below is removed to stop broadcasting decided messages ----------

	// if err := c.broadcastDecided(decidedMsg); err != nil {
	// 	// no need to fail processing instance deciding if failed to save/ broadcast
	// 	fmt.Printf("%s\n", err.Error())
	// }
	// return decidedMsg, nil
}

```

### Stage 2: Stop processing decided messages
Operators will also stop processing decided messages. All codes related to decided messages will be removed.

## Backwards Compatibility
It is backwards compatible to implement the first stage of changes of stop broadcasting decided messages, but it is not backwards compatible to implement the second stage of changes of stop processing decided messages.

The first stage of changes can be implemented in a non-forked update and the second stage of changes can be implemented in a fork.

But even if a decided message is received after the update, it will not damage the security or liveness of the consensus.

