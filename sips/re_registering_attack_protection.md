| Author                      | Title                           | Category | Status              |
|-----------------------------| ------------------------------- | -------- | ------------------- |
| Lior Rutenberg (@lior-blox) | SSV Register Cluster copied shares attack | Core     | open-for-discussion |

## Summary

A "copied shares" attack can occur when a malicious actor registers valid shares to a legitimate validator by either "frontrunning" a valid registration of a cluster transaction, or re-registering a previously removed validator.

In both scenarios, a validator would be operating on the SSV network under the control of a harmful user instead of the rightful validator owner.

#### Front-Running
In this scenario, a malicious actor could "front-run" a genuine user trying to register a validator. The registration would appear as legitimate. To keep the attack ongoing, the attacker would cover the validator's fees.

To make any changes (such as removing or changing the cluster), the genuine user would need to exit the validator.

#### Re-registering attack
In this case, an attacker could identify a validator that has been removed (from the SSV network) and quickly re-register it, thereby preventing the owner from transferring the validator elsewhere. Similar to the front-running scenario, the attacker would cover the validator's fees to keep the attack active.

In response, the genuine user would need to exit the validator immediately.

## Rational & Design Goals
This SIP proposes a method to safeguard users by preventing unauthorized access to their validators by harmful entities. The proposal ensures that only the legitimate owner of a validator can enroll it in the SSV network. The process involves using the validator's private key to sign the address that is registering, along with a nonce.

The nonce is a necessary addition to prevent a unique edge case where, if a validator address that has been removed is compromised, a malicious party could potentially re-register it using the original address.

The BLS signature's authenticity will be confirmed by the SSV nodes, which will use the validator's public key and ensure that the nonce value is greater than the previous one. Instead of being on a validator level, the nonce will be on an account level to prevent scenarios where a validator is deregistered and re-registered by the same user.

The decision regarding whether the nonce should be stored on the contract or solely on the SSV nodes still needs to be made.

## Specification
The specification is organized into three distinct sections:

1. Modifying the validator enrollment procedure to incorporate the validator's signature on the address.
2. Altering the validator map keys within the contract to include the address along with the public key.
3. Implementing signature verification within the SSV nodes during the validator registration process.

> Please be aware: A new contract deployment will be necessary to modify the validator map.
> Furthermore, the uniqueness of a public key will be associated with each individual address, rather than being unique across the entire contract.

### SSV Keys
Inputs for SSV keys should encompass both the validator's address and the most recent account nonce.

These inputs will utilize the validator's private key to sign the address as well as the incremented nonce.

The resultant output should not only contain the current output but also the signature and the incremented nonce.

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
> Note: The input parameters for the registerValidator function are not yet finalized and should be evaluated in light of optimizing gas fees. 
>
> Several potential arrangements are under consideration:
> 1. Consolidating shares, signature, and nonce into a single bytes array.
> 2. Merging signature and nonce into a single bytes array.
> 3. Keeping all elements separate.


