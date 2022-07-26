| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| Alon Muroch | Messages structure and encoding | Core     | draft  |

**Summary**  
Describes consensus and post consensus message structure and encoding for the SSV.Network.
The structure minimizes message relaying.
Encoding chosen is SSZ.

**Rational & Design Goals**  
SSV produces, at a minimum, 10 new messages (proposal, 3X prepare, 3X commit, 3X partial signature) per validator attestation per epoch that are propagated in the SSV network. Since every validator has 1 attestation per epoch (other duties are significantly rarer) it’s obvious that attestations make the bulk of the SSV’s network traffic.

With that in mind the following SIP focuses on message structure to accomplish:
* Minimize the number of messages being broadcasted
* Simplify the number of message types
* Minimize message size on wire (including use of compression)
* Standardize using known encoding formats

**Specification**

**Encoding**  
In the sake of simplicity (and not re-inventing the wheel) we’ve taken a look at existing research done by ethereum for different encoding types [here](https://github.com/sigp/serialization_sandbox/blob/report/report/serialization_report.md) and [here](https://notes.ethereum.org/15_FcGc0Rq-GuxaBV5SP2Q?view).  
[SSZ](https://github.com/ethereum/consensus-specs/blob/dev/ssz/simple-serialize.md) was chosen as the encoding protocol for SSV messages because:
* Align with ethereum’s spec
* Deterministic message root
* Multiple implementations and spec tests
* Merkelized hash tree (used below)
* Encoding Size

Json is significantly more size efficient that the current json encoding used by SSV.
A simple size comparison test can be found [here](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/qbft/messages_test.go#L75-L83).  
SSZ is 66% smaller (in the test, 1,640 bytes vs 3,700) vs json, using snappy compression can reduce size by 95% to 197 bytes vs using snappy on a json encoding (548 bytes).

SSZ hash tree root enables substituting full data objects with their hash tree root without changing the full message hash tree root which also doesn’t change the BLS signature signing the object. Below we explain how this property is used.

**Message Types**  
There are 2 major message types: consensus and partially signed duty data.

**QBFT Messages**  
[Code](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/qbft/messages.go)  
QBFT messages represent all consensus messages sent on wire. We have 2 types of messages: regular and header messages.
Regular messages contain the full consensus input object in them, mainly used in proposal and round change messages.  
Header messages have just the hash tree root for the consensus input object to save bandwidth.  
SSZ’s properties allow this substitution without changing the actual signature, we can even move (one way) from SignedMessage to SignedMessageHeader

```go
// Message includes the full consensus input to be decided on, used for proposal and round-change messages
type Message struct {
	ID     types.MessageID `ssz-size:"52"`
	Type   Type
	Height uint64
	Round  uint64
	Input  types.ConsensusInput
	// PreparedRound an optional field used for round-change
	PreparedRound uint64
}

// SignedMessage includes a signature over Message AND optional justification fields (not signed over)
type SignedMessage struct {
        Message   Message
        Signers   []uint64 `ssz-max:"13"`
        Signature [96]byte `ssz-size:"96"`

        RoundChangeJustifications []*SignedMessageHeader `ssz-max:"13"`
        ProposalJustifications    []*SignedMessageHeader `ssz-max:"13"`
}

// MessageHeader includes just the root of the input to be decided on (to save space), used for prepare and commit messages
type MessageHeader struct {
        ID            types.MessageID `ssz-size:"52"`
        Type          Type
        Height        uint64
        Round         uint64
        InputRoot     [32]byte `ssz-size:"32"`
        PreparedRound uint64
}

// SignedMessageHeader includes a signature over MessageHeader
type SignedMessageHeader struct {
        Message   MessageHeader
        Signers   []uint64 `ssz-max:"13"`
        Signature [96]byte `ssz-size:"96"`
}
```
SignedMessageHeader structs do not contain justification fields per QBFT form [specification](https://entethalliance.github.io/client-spec/qbft_dafny_spec/types.dfy)

**Partially Signed Duty Messages**  
[Code](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/ssv/messages.go)  

```go
type PartialSignature struct {
	Slot          uint64
	Signature     [96]byte `ssz-size:"96"`
	SigningRoot   [32]byte `ssz-size:"32"`
	Signer        uint64
	Justification *qbft.SignedMessageHeader
}

type SignedPartialSignatures struct {
	ID                types.MessageID     `ssz-size:"52"`
	PartialSignatures []*PartialSignature `ssz-max:"13"`
}
```

**MessageID Quick Traversing**  
The SSV node checks for message ID to route incoming messages for processing, discarding irrelevant messages. Message ID is constructed from the [validator’s public key and beacon role](https://github.com/bloxapp/ssv-spec/blob/main/types/messages.go#L33-L53).  
SSZ enables quick traversal of items without decoding which makes message routing significantly faster and more efficient.
[Implementation](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/types/messages.go)
