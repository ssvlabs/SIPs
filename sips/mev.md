| Author                    | Title                            | Category | Status               |
|---------------------------|----------------------------------|----------|----------------------|
| Moshe Revah (@moshe-blox) | Support externally built blocks | Core     | open-for-discussion  |

## Summary
Support the production and proposal of blocks built by external entities, allowing operators to access blocks from a marketplace of builders such as [Flashbots](https://boost-relay.flashbots.net/).

## Rational & Design Goals
Block building is a computationally intensive task, and even more so when blockspace utilization and profit maximization are in mind. Additionally, many of the more profitable transactions, so-called MEV, are private and only accessible to validators through external block builders.

Currently, operators wishing to offer competitive returns must be sophisticated block builders with private transaction flows. This SIP aims to change that by allowing them to register with and propose blocks from external builders who comply to [ethereum/builder-specs](https://github.com/ethereum/builder-specs).

## Specification

### MEV-supporting operators

Operators who wish to support blinded blocks must configure their node to do so, otherwise their node will reject signing those.

Validators who want to propose MEV blocks would have to avoid registering with a combination of MEV and non-MEV operators, as the latter would reject any blinded blocks and result in a missed duty.

> TODO: can't MEV-supporting leaders just fall back to non-blinded blocks if at least one of the other operators is non-MEV supporting? This should allow such combinations.

### Validator registration

Upon registration of a validator to SSV, it's operators construct a `SignedValidatorRegistration` message with it's preferred `fee_recipient`, sign it, and finally submit it to their Beacon node with [/eth/v1/validator/register_validator](https://ethereum.github.io/beacon-APIs/#/Validator/registerValidator).

This prepares the Beacon node which also relays this information to it's configured builder(s).

Note: producing a `BlindedBeaconBlock` before and shortly thereafter (delay depends on the builder) the registration may result in either:

1. Beacon node falls back to locally-built block from it's execution layer.
2. Builder doesn't reward the validator's `fee_recipient` because it isn't aware of it, or rewards a potentially different `fee_recipient` from the validator's latest registration.

> TODO: if we choose to repeat this periodically, it might make it redundant to do so upon validator registration as well?

### Blinded block proposals

#### Scenario A: all operators share the same builders

1. Leader produces a `BlindedBeaconBlock` from their Beacon node with [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock) and broadcasts it.

2. Operators sign the `BlindedBeaconBlock`.

3. Operators publish the `SignedBlindedBeaconBlock` to their Beacon node with [/eth/v1/beacon/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlindedBlock).
  
    The Beacon node reveals the signature to the builder, which (ideally) reveals the transactions in return, after which it unblinds the block and publishes it to the network.

    ##### Issues

    Unrealistic?

#### Secnario B: operators bypass the Beacon node to unblind from builders directly

1. Leader produces a `BlindedBeaconBlock` from their Beacon node with [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock) and broadcasts it together with the endpoint of it's builder as `BuilderBlindedBeaconBlock`.

    ```go
    type BuilderBlindedBeaconBlock {
        BuilderEndpoint    string
        BlindedBeaconBlock *spec.VersionedBlindedBeaconBlock
    }
    ```

2. Operators sign the `BlindedBeaconBlock`.

3. Operators submit the `SignedBlindedBeaconBlock` to the builder's endpoint with [/eth/v1/builder/blinded_blocks](https://ethereum.github.io/builder-specs/#/Builder/submitBlindedBlock) to reveal the transactions and unblind the block into a `SignedBlindedBlock`.

4. Operators publish the `SignedBeaconBlock` to their Beacon node with [/eth/v1/beacon/blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlock).

    ##### Issues

    ###### Unknown builder
    When producing blinded blocks with a Beacon node connected to multiple builders (currently possible via mev-boost), the block's builder is unspecified. So the leader can't determine which builder endpoint to share with the other operators, and only he can unblind & publish it.

    We can mitigate this by requiring leaders to share the full list of builders they're working with, so that the other operators can try to unblind with each of them (ideally in parallel).