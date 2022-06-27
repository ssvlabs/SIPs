| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| Alon Muroch | Generalized DKG support in SSV | Core     | Draft  |

**Summary**
Describes a general workflow and specifications for integrating DKG into SSV regardless of the DKG protocol selected, number of rounds it requires or data structures it uses.
This SIP describes initializing, running and collecting the output of a DKG protocol.

Rational & Design Goals
This SIP does not change the way SSV validators are registered (on-chain via contract) or managed but rather just the way they are created (their private/ public keys).
The party initializing the DKG does not necessarily take part in the DKG.
The incentives for operators to participate in a DKG ceremony are the potential fees that can be collected from the DKG validator registering.
Use cases for DKG can range from on-chain staking pools to custodial institutional stakers. The DKG protocol should not be tightly coupled with the way the resulting validator keys are used.

**Specification**

**Networking**  
DKG between independent parties requires P2P communication, SSV’s network is a pub-sub network that does not couple operators and IP addresses.
We add a dedicated pub-sub topic called ssv.dkg where all DKG communication happens.
All SSV nodes register to that topic.

**Messages**
New SSV MsgType is introduced

| Name       | Type    | Value | Description                          |
|------------|---------|-------|--------------------------------------|
| DKGMsgType | MsgType | 3     | SSVMessage type for all DKG messages |

The SSVMessage Data field will encode a DKG SignedMessage data structure. 

```go
type Message struct {
MsgType MsgType
Identifier    RequestID
Data    []byte
}

type SignedMessage struct {
Message   *Message
Signer    types.OperatorID
Signature types.Signature
}
```

DKG Message types

| Name               | Type    | Value | Description                   |
|--------------------|---------|-------|-------------------------------|
| InitMsgType        | MsgType | 0     | Initialize DKG                |
| ProtocolMsgType    | MsgType | 1     | All DKG protocol msgs         |
| DepositDataMsgType | MsgType | 2     | Partial deposit data msg type |

The Data structure in the DKG message struct encodes 2 mandatory structs (Init and Output, see below) and other implementation specific structs which are out of scope for this SIP.

**RequestID**
Is a unique identifier for the entire DKG process which is a combination of the signing eth address of the message and an incremental index.

Requires Message.ID.GetETHAddress() == singing key

**Signing key**
Messages are signed with ethereum’s ECDSA keys to be compatible with smart contracts and also enable anyone to run DKG.
See signing for more details

**Initializing a DKG**
Any peer on the network can request a DKG to initialize by sending a DKG init message to the DKG topic.

```go
// Init is the first message in a DKG which initiates a DKG
type Init struct {
// OperatorIDs are the operators selected for the DKG
OperatorIDs []types.OperatorID
// Threshold DKG threshold for signature reconstruction
Threshold uint16
// WithdrawalCredentials used when signing the deposit data
WithdrawalCredentials []byte
// Fork is eth2 fork version
Fork phase0.Version
}

```

Every operator with id in OperatorIDs participates in the DKG protocol, broadcasting messages to the DKG topic.
The withdrawal credentials are configured according to the eth consensus spec

**DKG Output**
A DKG output, the result of a fully executed DKG, must be encoded and signed with the signing key.
We sign it in such a way so any smart contract can verify the operator actually produced it and not a malicious actor.
The operators themselves can act maliciously, the DKG requester should pick his operators in a diligent way,

The output includes all crucial data for a successful deposit and operation of a validator. It assumes an honest majority between the selected DKG operators.

```go
// Output is the last message in every DKG which marks a specific node's end of process
type Output struct {
// RequestID for the DKG instance (not used for signing)
RequestID RequestID
// EncryptedShare standard SSV encrypted shares
EncryptedShare []byte
// SharePubKey is the share's BLS pubkey
SharePubKey []byte
// ValidatorPubKey the resulting public key corresponding to the shared private key
ValidatorPubKey types.ValidatorPK
// DepositDataSignature reconstructed signature of DepositMessage according to eth2 spec
DepositDataSignature types.Signature
}

type SignedOutput struct {
// Data signed
Data *Output
// Signer operator ID which signed
Signer types.OperatorID
// Signature over Data.GetRoot()
Signature types.Signature
}

```

The DKGOutput object must have a GetRoot() ([]byte, error) function which encodes the struct with abi.encode and returns the abi encoded struct with a keccak256 hash function.
See example [here](https://gist.github.com/alonmuroch/38a7c4f3360887e6aebde0cdc3d82fc8).

SignedOutput holds an ECDSA signature over Output.GetRoot

```go
func SignOutput(output *Output, privKey *ecdsa.PrivateKey) (types.Signature, error) {
root, err := output.GetRoot()
if err != nil {
return nil, errors.Wrap(err, "could not get root from output message")
}
return crypto.Sign(root, privKey)
}
```

The Output struct signed with the signing key must be the last message sent by each of the operators for a successful DKG to be concluded. 

```go
type SignedDKGOutput struct {
// Data signed
Data *DKGOutput
// Signer operator ID which signed
Signer OperatorID
// Signature over Data.GetRoot()
Signature Signature
}

```

Any user can now take the encrypted shares and validator pub key and register the validator in the SSV contracts