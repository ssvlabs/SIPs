| Author      | Title                          | Category | Status   |
|-------------|--------------------------------|----------|----------|
| Alon Muroch | Messages structure and encoding | Core     | draft v2 |

**Summary**  
Describes consensus and post consensus message structure and encoding for the SSV.Network.
The structure minimizes message relaying.
Encoding chosen is SSZ.  

[Ref implementation](https://github.com/bloxapp/ssv-experiments/tree/master/ssz_encoding)

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

**Wire Message**  
A wire message struct has the responsibility of encapsulating an encoded and compresses SSZ structs. 
It also has message ID and type encoded in it for easy traversal so SSV nodes could quickly identify where to route the message for processing

Message types are now 4 byte arrays for easy and efficient encoding

```go
type Message struct {
	ID            MessageID `ssz-size:"32"`
	DataSSZSnappy []byte    `ssz-max:"2048"`
}
```

**Message ID**  
Message ID is a 32 byte array which can take different uses for different message types.  
For consensus, decided and partial signature messages the structure is:   

| Validator Index | Beacon Role | Padding  | Message Type |
|-----------------|-------------|----------|--------------|
| 8 bytes         | 4 bytes     | 16 bytes | 4 bytes      |


For DKG messages the structure is: 

| ETH Address | Index   | Padding | Message Type |
|-------------|---------|---------|--------------|
| 20 bytes    | 4 bytes | 4 bytes | 4 bytes      |

**Message Types**  
There are 3 major message types: consensus partially signed duty data and DKG.
For easy traversal without decompression and decoding, the message types include sub types.  
Consensus messages start with 0x1 and have 4 subtypes.  
The rest have incremental first byte.

```go
var (
    // ConsensusProposeMsgType QBFT propose consensus message
    ConsensusProposeMsgType = MsgType{0x1, 0x0, 0x0, 0x0}
    // ConsensusPrepareMsgType QBFT prepare consensus message
    ConsensusPrepareMsgType = MsgType{0x1, 0x1, 0x0, 0x0}
    // ConsensusCommitMsgType QBFT commit consensus message
    ConsensusCommitMsgType = MsgType{0x1, 0x2, 0x0, 0x0}
    // ConsensusRoundChangeMsgType QBFT round change consensus message
    ConsensusRoundChangeMsgType = MsgType{0x1, 0x3, 0x0, 0x0}
    
    // DecidedMsgType are all QBFT decided messages
    DecidedMsgType = MsgType{0x2, 0x0, 0x0, 0x0}
    
    // PartialSignatureMsgType are all partial signatures msgs over beacon chain specific signatures
    PartialSignatureMsgType = MsgType{0x3, 0x0, 0x0, 0x0}
    
    // DKGInitMsgType sent when DKG instance is started by requester
    DKGInitMsgType = MsgType{0x4, 0x0, 0x0, 0x0}
    // DKGProtocolMsgType contains all key generation protocol msgs
    DKGProtocolMsgType = MsgType{0x4, 0x1, 0x0, 0x0}
    // DKGDepositDataMsgType post DKG deposit data signatures
    DKGDepositDataMsgType = MsgType{0x4, 0x2, 0x0, 0x0}
    // DKGOutputMsgType final output msg used by requester to make deposits and register validator with SSV
    DKGOutputMsgType = MsgType{0x4, 0x3, 0x0, 0x0}
    
    // UnknownMsgType can't be identified
    UnknownMsgType = MsgType{0x0, 0x0, 0x0, 0x0}
)
```

**Quick Traversing**  
The SSV node checks for message ID and type to route incoming messages for processing, discarding irrelevant messages. Message ID is constructed from the validator’s index and beacon role.  
SSZ enables quick traversal of items without decoding which makes message routing significantly faster and more efficient.
[Implementation](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/types/message_id.go#L36-L54)


**QBFT Messages**  
[Code](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/qbft/messages.go)  
QBFT messages represent all consensus messages sent on wire. We have 2 types of messages: regular and header messages.
Regular messages contain the full consensus input object in them, mainly used in proposal and round change messages.  
Header messages have just the hash tree root for the consensus input object to save bandwidth.  
SSZ’s properties allow this substitution without changing the actual signature, we can even move (one way) from SignedMessage to SignedMessageHeader.  

Notice that there are no message ID and type in the QBFT message, those types are move to the outer Message struct.

SignedMessageHeader structs do not contain justification fields per QBFT form [specification](https://entethalliance.github.io/client-spec/qbft_dafny_spec/types.dfy)

**Partially Signed Duty Messages**  
[Code](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/ssv/messages.go)  

**DKG Messages**  
[Code](https://github.com/bloxapp/ssv-experiments/blob/master/ssz_encoding/dkg/messages.go)  
