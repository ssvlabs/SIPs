| Author                    | Title                           | Category | Status              |
| ------------------------- | ------------------------------- | -------- | ------------------- |
| Moshe Revah (@moshe-blox) | Support externally built blocks | Core     | open-for-discussion |

## Summary

Support the production and proposal of blocks built by external entities, allowing operators to access blocks from a marketplace of builders such as [Flashbots](https://boost-relay.flashbots.net/).

## Rational & Design Goals

Block building is a computationally intensive task, and even more so when blockspace utilization and profit maximization are in mind. Additionally, many of the more profitable transactions, so-called MEV, are private and only accessible to validators through external block builders.

Currently, operators wishing to offer competitive returns must be sophisticated block builders with private transaction flows. This SIP aims to change that by allowing them to register with and propose blocks from external builders who comply to [ethereum/builder-specs](https://github.com/ethereum/builder-specs).

## Specification

### Configuration

Operators who wish to propose blinded blocks must configure their node to do so, otherwise it would only propose standard blocks and reject QBFT proposals containing blinded blocks.

Therefore, validators should solely select operators with matching configuration, otherwise consensus might be unreachable.

### Validator registration

Builders require validators to publish a [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration) to set their `fee_recipient` and `gas_limit` preferences.

When building a block for a validator, builders refer only to the registration with the highest timestamp.

In the wild, some Ethereum validator clients currently publish this message every epoch.

#### Fee recipients

Validators may set their preferred `fee_receipient` address by calling `setFeeRecipientAddress` in the `SSVNetwork` contract. Validators may repeat this call as their preference changes over time.

#### Signing

At the start of every slot, operators select their active validators with `ShouldRegisterValidatorAtSlot`, and for those, produce a [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration) with their preferred `fee_recipient`.

```go
ValidatorRegistrationSlotInterval = 10 * SlotsPerEpoch

func ShouldRegisterValidatorAtSlot(index phase0.ValidatorIndex, slot phase0.Slot) bool {
    return (index + slot) % ValidatorRegistrationSlotInterval == 0
}
```

#### Publishing

Every slot, operators should publish the most recent successfully signed registration for `1/SlotsPerEpoch` of their validators, so that all validators are registered at least once per epoch.

> Note: Since relays/builders can have some delay with registrations, producing a blinded block before and shortly thereafter publishing the first registration may result in either:
>
> 1. Builder is unfamiliar with the validator yet and doesn't build a block, and Beacon node falls back to locally-built block from it's execution layer.
> 2. Builder doesn't reward the validator's `fee_recipient` because it isn't aware of it yet.
> 3. Builder rewards a potentially different `fee_recipient` from the validator's latest registration (such as a registration prior to onboarding to SSV.)

> Note: Currently, implementation publishes registration differently: right after producing signatures. We're following the performance of that, and will either revise this SIP or the implementation accordingly.

#### Issue: Gas limits

Gas limits are set by operators rather than validators, unlike in standard validator clients.

This SIP proposes to hardcode the gas limit to 30 million (which is the default in Prysm and Lighthouse), but recommends to keep watching it and modify if necessary.

### Blinded block proposals

When a validator has a proposal duty, their operators:

1. Produce a `BlindedBeaconBlock` from their Beacon node at [/eth/v1/validator/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Validator/produceBlindedBlock)
2. Reach consensus on and sign the round leader's `BlindedBeaconBlock`
3. Submit the `SignedBlindedBeaconBlock` to their Beacon nodes at [/eth/v1/beacon/blinded_blocks](https://ethereum.github.io/beacon-APIs/#/Beacon/publishBlindedBlock)

Under the hood, each operator's Beacon node would attempt to unblind the block from it's connected builder(s) by revealing the signature to them, which would fail if they haven't built or don't know of this block.

Therefore, for unblinding to succeed, operators' Beacon nodes must share the same builder(s) with the round leader's Beacon node.

Since SSV can't enforce a specific set of builder(s) at the protocol-level, operators would have to reach social consensus on which builder(s) to configure their Beacon nodes with.

#### Locally-built blinded blocks

Sometimes, a Beacon node falls back to a locally-built `BlindedBeaconBlock` from it's execution layer, either because it's configured to do so under certain conditions (profitability threshold; chain is unstable) or because the builder(s) are unavailable.

A locally-built blinded block can only be unblinded by the same Beacon node which built it, which unfortunately means that only the round leader would successfully submit the block, thereby reducing the total network outreach of the proposal down to a single Beacon node.

We should consider pushing operators to configure their node for less harsh fallback conditions, so that more of their blocks are externally-built and can be successfully submitted by other operators' Beacon nodes.

#### Blinded block validation

Nowadays, there's no meaningful validation we can do on blinded blocks, but there's [active discussion](https://github.com/flashbots/mev-boost/issues/99) about requiring builders to attach payment proofs on bids so that they can't cheat and reward themselves instead of the validator's `fee_recipient`.

Ideally, payment proof validation should be handled by [mev-boost](https://github.com/flashbots/mev-boost), as it already examines the builders' bids, in which case operators would only have to ensure their Beacon nodes are always updated with the latest [`builder-specs/SignedValidatorRegistration`](https://ethereum.github.io/builder-specs/#model-SignedValidatorRegistration).
