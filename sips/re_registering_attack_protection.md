| Author                      | Title                           | Category | Status              |
|-----------------------------| ------------------------------- | -------- | ------------------- |
| Lior Rutenberg (@lior-blox) | SSV Register Cluster copied shares attack | Core     | open-for-discussion |

## Summary

A copied shares attack can happen by an attacker registering valid shares to a valid validator by frontrunning a valid register cluster transaction or re-registering a removed validator.

In both cases a validator will be running on SSV by a malicious user instead of the owner of the validator.

#### Front-Running
A malicious attacker will front-run an honest user registering a validator.
The registration will pass as valid.
The attacker will pay fees for the validator to maintain the attack active.

The honest user will need to exit the validator to make any change (remove/ change cluster).

#### Re-registering attack
An attacker will detect a removed validator (removed from the SSV network) and will re-register it immediately, preventing the owner from moving the validator away.
The attacker will pay fees for the validator to maintain the attack active. 

The honest user will need to exit the validator immediately.

## Rational & Design Goals
We need to protect users from having their validators being hijacked by malicious actors by ensuring that only the owner of the validator can register it to the SSV network.
This SIP proposes a solution using validator private key to sign on the address that making the registration and a nonce.
A nonce is required to prevent re-registering edge case where the removed validator address is compromised and the attacker can re-register it with the original address. 
The BLS signature will be verified by the ssv nodes using the validator public key and making sure the nonce is higher than the last one.
The nonce will be on an account level and not a validator level to prevent the case where a validator is removed and re-registered by the same user.
It should be decided if it's needed to be stored on the contract or on the SSV nodes only.


## Specification
The specification is divided into 3 parts:
1. Changing the validator registration process to include the validator's signature on the address.
2. Changing the validator map keys in the contract to include address in addition to the public key.
3. Adding the signature verification to the SSV nodes upon validator registration.


### SSV Keys
SSV keys should input should include the validator's address and the last account nonce.
It will use the validator private key to sign on the address and the incremented nonce.
The output should include the signature and the incremented nonce in addition to the current output.

### SSV Contract

Changing the validator mapping to include the address in addition to the public key.

From:
```solidity
    struct Validator {
        address owner;
        bool active;
    }
    mapping(bytes32 => Validator) public validatorPKs;
```
To: 
```solidity
    struct Validator {
        bytes32 hashedOperators;
        bool active;
    }
    mapping(bytes32 => Validator) public validatorPKs;
    
    //example of setting the mapping
    validatorPKs[keccak256(abi.encodePacked(publicKey, msg.sender))] = Validator({hashedOperators: keccak256(operatorIds), active: true});
    
    function registerValidator(
        bytes calldata publicKey,
        uint64[] memory operatorIds,
        bytes calldata shares,
        bytes calldata signature, //signature on the address and nonce
        uint256 nonce,
        uint256 amount,
        Cluster memory cluster
    )
    
```
> Note: registerValidator function input parameters are not final and should consider gas fees optimisation. 
>
> 1. Combined shares, signature and nonce to one bytes array. 
> 2. Combined signature and nonce to one bytes array.
> 3. Keep the all separate.

