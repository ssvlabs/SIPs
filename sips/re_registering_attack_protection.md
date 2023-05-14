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
This SIP proposes a method to safeguard users by preventing unauthorized access to their validators by harmful entities. The proposal ensures that only the legitimate owner of a validator can enroll it in the SSV network. The process involves using the validator's private key to sign the address that is registering, along with a nonce.

The nonce is a necessary addition to prevent a unique edge case where, if a validator address that has been removed is compromised, a malicious party could potentially re-register it using the original address.

The BLS signature's authenticity will be confirmed by the SSV nodes, which will use the validator's public key and ensure that the nonce value is greater than the previous one. Instead of being on a validator level, the nonce will be on an account level to prevent scenarios where a validator is deregistered and re-registered by the same user.

The decision regarding whether the nonce should be stored on the contract or solely on the SSV nodes still needs to be made.

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

