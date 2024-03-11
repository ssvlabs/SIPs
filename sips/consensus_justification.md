|     Author     |          Title          | Category |       Status        |    Date    |
| -------------- | ----------------------- | -------- | ------------------- | ---------- |
| Matheus Franco | Consensus Justification | Core     | open-for-discussion | 2024-03-08 |

## Summary

Drop decided messages and include a consensus justification nested in post-consensus messages.

## Motivation

Decided messages represent $16$% of the exchanged messages in the network and also significantly increase the thresholds of allowed messages in message validation. Furthermore, it's not a necessary part of the protocol and the speed-up that it provides can be provided by a simpler justification in the post-consensus message.

## Rationale

As it's done with the pre-consensus justification, that is included in the *Proposal* message, we propose including a consensus justification (an aggregated commit message) in the post-consensus message instead of sending decided messages.

First of all, the decided messages are useful to allow nodes to start the post-consensus phase if they are "behind" in the protocol execution due to not receiving messages. However, a consensus justification inside a post-consensus message allows a node to accomplish the same result while progressing even further due to the post-consensus signature. Moreover, this proposal aligns the design decisions of the (pre-consensus, consensus) and (consensus, post-consensus) inter-phases.

## Improvements

This proposal reduces the number of messages exchanged in the network by $16$%.

## Spec change

Within a `SignedPartialSignatureMessage.PartialSignatureMessages`, all `PartialSignatureMessage` objects refer to the same `QBFT Instance`. Thus, we can simply add the justification to the `SignedPartialSignatureMessage` as follows:

```go
type SignedPartialSignatureMessage struct {
	Message   PartialSignatureMessages
	Signature Signature `ssz-size:"96"`
	Signer    OperatorID
    Justification *SignedMessage // Aggregated commit message (or []*SignedMessage if we ever use another asymmetric scheme that doesn't allow aggregation)
}
```

It could also be added to the `PartialSignatureMessages` types. The difference is that this would *hold* the `SignedPartialSignatureMessage.Signature` also to the consensus justification.


## Drawbacks

- The post-consensus messages would be bigger due to the aggregated commit message.
- Exporters use decided messages to compute statistics of the network. Therefore, the proposal implies not only a protocol change but also changes in the exporter implementation.