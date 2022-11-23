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

#### Metadata
Operators advertise whether they intend to build blinded blocks and which MEV relays they work with.

Since blinded blocks can't be validated for their `fee_recipient`, some validators may be willing to compromise on potential profits from MEV in return for the guaranteed block reward.

#### Node behaviour
Operators who wish to support blinded blocks must configure their node to do so, otherwise their node will reject signing those.

Validators who want to propose MEV blocks would have to avoid registering with a combination of MEV and non-MEV operators, as the latter would reject any blinded blocks and result in a missed duty.

> TODO: can't leaders just fall back to propose non-blinded blocks if at least one of the other operators is non-MEV supporting? This should allow such combinations.

### Validator registration

When a validator registers to the SSV contract, he may attach a [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration) including his `fee_receipient` and `gas_limit` preferences.

The SSV contract must allow validators to update this attachment, as their preferences may change over time.

Operators must validate the signature and keep track of their validators' latest attachment (by highest `timestamp`), and periodically publish it to their Beacon nodes at [/eth/v1/validator/register_validator](https://ethereum.github.io/beacon-APIs/#/Validator/registerValidator).

An operator may publish for his entire validator set in bulk, however a single invalid signature may cause the request to fail and other attachments to be ignored, so a prior signature validation is crucial.

> Note: producing a `BlindedBeaconBlock` before and shortly thereafter (delay depends on the builder) the registration may result in either:
> 1. Beacon node falls back to locally-built block from it's execution layer.
> 2. Builder doesn't reward the validator's `fee_recipient` because it isn't aware of it.
> 3. Builder rewards a potentially different `fee_recipient` from the validator's latest registration (such as a registration prior to onboarding to SSV.)

#### Issues

##### Gas limit

Within the proposed construction, the gas limit is set by validators only and relies on them to update it if needed.

If gas limit is likely to change to adjust to network conditions, then this isn't ideal because validators would have to track of it, and when it changes — produce a new signature and publish an update transaction to the contract (which may be very expensive when the network is congested).

> TODO: examine the code of validator clients such as Lighthouse and Prysm to figure this out.

### Blinded block proposals

#### Scenario A — Block is published only by operators who share builder with leader

1. Leader produces a `BlindedBeaconBlock` from their Beacon node at [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock) and broadcasts it.

2. Operators sign the `BlindedBeaconBlock`.

3. Operators publish the `SignedBlindedBeaconBlock` to their Beacon node at [/eth/v1/beacon/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlindedBlock).
  
    The Beacon node reveals the signature to the builder, which (ideally) reveals the transactions in return, after which it unblinds the block and publishes it to the network.

    ##### Issues

    Unrealistic?

#### Secnario B — Block is published by all operators

1. Leader produces a `BlindedBeaconBlock` from their Beacon node at [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock) and broadcasts it together with the endpoint of it's builder as `BuilderBlindedBeaconBlock`.

    ```go
    type BuilderBlindedBeaconBlock {
        BuilderEndpoint    string
        BlindedBeaconBlock *spec.VersionedBlindedBeaconBlock
    }
    ```

2. Operators sign the `BlindedBeaconBlock`.

3. Operators submit the `SignedBlindedBeaconBlock` to the builder's endpoint at [/eth/v1/builder/blinded_blocks](https://ethereum.github.io/builder-specs/#/Builder/submitBlindedBlock) to reveal the transactions and unblind the block into a `SignedBlindedBlock`.

4. Operators publish the `SignedBeaconBlock` to their Beacon node at [/eth/v1/beacon/blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlock).

    ##### Issues

    ###### Unknown builder
    When producing blinded blocks with a Beacon node connected to multiple builders (currently possible via mev-boost), the block's builder is unspecified. So the leader can't determine which builder endpoint to share with the other operators, and only he can unblind & publish it.

    We can mitigate this by requiring leaders to share the full list of builders they're working with, so that the other operators can try to unblind with each of them (ideally in parallel).

#### Scenario C — Leader publishes revealed transactions for others to submit

This increases both network traffic and delay to publish, so we'll probably not do it.

#### Blinded block validation

Nowadays, there's no meaningful validation we can do on blinded blocks, but there's [active discussion](https://github.com/flashbots/mev-boost/issues/99) about requiring builders to attach payment proofs on bids so that they can't cheat and reward themselves instead of the validator's `fee_recipient`.

Ideally, payment proof validation should be handled by mev-boost, as it already examines the builders' bids, in which case operators would only have to ensure their Beacon nodes are always updated with the latest [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration).