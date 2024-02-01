|     Author     |        Title         | Category |       Status        |
| -------------- | -------------------- | -------- | ------------------- |
| Matheus Franco | Formal Spec Messages | Core     | open-for-discussion |

## Summary

The current implementation of the QBFT protocol has $\mathcal{O}(n^3)$ **communication complexity** (number of bits exchanged until termination). The reason is that the _Proposal_ message contains nested _Round-Change_ messages and each _Round-Change_ message contains nested _Prepare_ messages.

On the other hand, the [QBFT formal-spec](https://github.com/Consensys/qbft-formal-spec-and-verification/tree/main) doesn't include the Round-Change justification component (_Prepare_ messages) in the nested _Round-Change_ messages inside a _Proposal_, achieving  $\mathcal{O}(n^2)$ complexity. Thus, we propose aligning the current implementation with the Dafny specification.

## Motivation

Change the $\mathcal{O}(n^3)$ communication complexity to $\mathcal{O}(n^2)$ by changing the message structures.

## Rationale

The changes are guided by the [QBFT formal spec](https://github.com/Consensys/qbft-formal-spec-and-verification/tree/main).


## Current state

The current QBFT message structures are depicted below.

```go
type Message struct {
	MsgType    MessageType
	Height     Height // QBFT instance Height
	Round      Round  // QBFT round for which the msg is for
	Identifier []byte // instance Identifier this msg belongs to

	Root                     [32]byte
	DataRound                Round    // The last round that obtained a Prepare quorum
	RoundChangeJustification [][]byte
	PrepareJustification     [][]byte
}
type SignedMessage struct {
	Signature   types.Signature
	Signers     []types.OperatorID
	Message     Message

	FullData    []byte
}
```

## Specification

We propose the following new message structures.

```go
type Message struct {
	MsgType    MessageType
	Height     Height // QBFT instance Height
	Round      Round  // QBFT round for which the msg is for
	Identifier []byte // The instance Identifier this msg belongs to

	Root          [32]byte // Proposed value Root
	PreparedRound Round    // Last prepared round
	PreparedValue [32]byte // Last prepared value
}

type SignedMessage struct {
	Signature types.Signature // Signature from Signers over Message
	Signers   []types.OperatorID
	Message   Message
}

type QBFTMessage struct {
	SignedMessage            SignedMessage
	FullData                 []byte          // Proposed consensus data
	ProposalJustification    []SignedMessage // Set of Round-Change messages that justifies a Proposal
	RoundChangeJustification []SignedMessage // Set of Prepare messages that justifies a prepared Round-Change
}
```
