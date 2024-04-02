|     Author     |           Title           | Category |       Status        |    Date    |
| -------------- | ------------------------- | -------- | ------------------- | ---------- |
| Matheus Franco | QBFT - Drop redundant BLS | Core     | open-for-discussion | 2024-03-27 |

[Discussion](https://github.com/bloxapp/SIPs/discussions/38)

## Summary

Add an RSA signature check in the Validator level and drop the consequent redundant BLS verifications in QBFT.

## Motivation

BLS verification is the most costly operation in the protocol. Thus, for scalability purposes, it's fundamental to search for faster alternatives to authenticate a message.

## Rationale

A message can be authenticated using a signature created with the operator's network public RSA key. This authentication method is considerably faster the authenticating with BLS. Thus, we can add in the `Validator.ProcessMessage` layer, an RSA check to ensure the authenticity of the received message. Notice that this check has an equivalent purpose to the BLS check in the `SignedMessage` structure. Thus, the BLS check becomes redundant.

Nonetheless, note that the BLS verification must be performed for the following two cases:
- messages contained in justifications
- decided messages

This is due to the fact that the contents of such messages are signed by different operators. So, we can't rely on the RSA signature to validate them.

## Improvements

For the attestation duty case (the most frequent one), the new cryptography cost is reduced to $62$% of the current value, a 1.6x boost.

The cryptography costs of the duty's steps are shown below.


<p align="center">
<img src="./images/qbft_drop_redundant_bls/redundant_bls_qbft_gantt.png"  width="30%" height="10%">
</p>


## Spec change

### QBFT

The `BaseValidation` functions of each message type can be split into a `NoVerification` and a `WithVerification` versions. The `WithVerification` calls the `NoVerification` and performs the BLS verification.

So, for example, the *commit* base validation would become:
```go
func baseCommitValidationWithVerification(
 	config IConfig,
 	signedCommit *SignedMessage,
 	height Height,
 	operators []*types.Operator,
 ) error {

 	if err := baseCommitValidationNoVerification(signedCommit, height, operators); err != nil {
 		return err
 	}

 	// verify signature
 	if err := signedCommit.Signature.VerifyByOperators(signedCommit, config.GetSignatureDomainType(), types.QBFTSignatureType, operators); err != nil {
 		return errors.Wrap(err, "msg signature invalid")
	}
	return nil
}


 func baseCommitValidationNoVerification(
 	signedCommit *SignedMessage,
 	height Height,
	operators []*types.Operator,
) error {

    // Similar to the original
	if signedCommit.Message.MsgType != CommitMsgType {
		return errors.New("commit msg type is wrong")
	}
	if signedCommit.Message.Height != height {
		return errors.New("wrong msg height")
	}
	if err := signedCommit.Validate(); err != nil {
 		return errors.Wrap(err, "signed commit invalid")
 	}

    // New
    // Add a check to confirm that the signer belongs to the committee
 	if !signedCommit.CheckSignersInCommittee(operators) {
 		return errors.New("signers not in committee")
 	}

    // No BLS Verification

 	return nil
}
```

The `CheckSignersInCommittee` can be defined in the following way:
```go
// Check if all signedMsg's signers belong to the given committee in O(n+m)
 func (signedMsg *SignedMessage) CheckSignersInCommittee(committee []*types.Operator) bool {
 	// Committee's operators map
 	committeeMap := make(map[uint64]struct{})
 	for _, operator := range committee {
 		committeeMap[operator.OperatorID] = struct{}{}
 	}

 	// Check that all message signers belong to the map
 	for _, signer := range signedMsg.Signers {
 		if _, ok := committeeMap[signer]; !ok {
 			return false
 		}
 	}
 	return true
}
```

Notice that these changes need to be implemented for:
- *prepare* messages: since they are received both in the raw version and nested into *round-change* and *proposal* justifications.
- *commit* messages: since they are received both in the raw version and aggregated into a *decided* message.
- *round-change* messages: since they are received both in the raw version and nested into a *proposal* justification.

The *proposal* base validation can just drop the BLS verification since it's never nested by any other messages.

The *Instance*'s `BaseMessageValidation` function should call the `NoVerification` version, while to validate justifications or decided messages, the ´WithVerification` version should be called.

### Validator

The `Validator`'s `ProcessMessage` function now receives a `SignedSSVMessage` instead of an `SSVMessage`. The `SignedSSVMessage` has an RSA signature which will be verified inside the `Validator.ProcessMessage`. If the signature is valid, the validator will call the duty runners processing functions for the nested `SSVMessage`. If the signature is invalid, it will return an error.

To handle the signature verification of `SignedSSVMessage` messages, the `Validator` will have a new `SignatureVerifier` interface.

```go
// SignatureVerifier is an interface responsible for the verification of SignedSSVMessages
type SignatureVerifier interface {
	// Verify verifies a SignedSSVMessage's signature using the necessary keys extracted from the list of Operators
	Verify(msg *SignedSSVMessage, operators []*Operator) error
}

type Validator struct {
	DutyRunners       DutyRunners
	Network           Network
	Beacon            BeaconNode
	Share             *types.Share
	Signer            types.KeyManager
	SignatureVerifier types.SignatureVerifier // New
}
```

The updated `ProcessMessage` function would be similar to:

```go

// ProcessMessage processes Network Message of all types
func (v *Validator) ProcessMessage(signedSSVMessage *types.SignedSSVMessage) error {

	// Validate message
	if err := signedSSVMessage.Validate(); err != nil {
		return errors.Wrap(err, "invalid SignedSSVMessage")
	}

	// Decode the nested SSVMessage
	msg := &types.SSVMessage{}
	if err := msg.Decode(signedSSVMessage.Data); err != nil {
		return errors.Wrap(err, "could not decode data into an SSVMessage")
	}

	// Verify SignedSSVMessage's signature
	if err := v.SignatureVerifier.Verify(signedSSVMessage, v.Share.Committee); err != nil {
		return errors.Wrap(err, "SignedSSVMessage has an invalid signature")
	}

	// Get runner
	// ...
}
```

This requires the `Operator` structure to hold a network public key. For that, we can change the `Operator` type in the following way:

```go
// Operator represents an SSV operator node
type Operator struct {
	OperatorID    OperatorID
	BeaconPubKey  []byte `ssz-size:"48"`
	NetworkPubKey []byte `ssz-size:"294"` // New
}

// GetNetworkPublicKey returns the network public key with which the node is identified with
func (n *Operator) GetNetworkPublicKey() []byte {
	return n.NetworkPubKey
}

```


## Drawbacks

No drawbacks could be found yet.