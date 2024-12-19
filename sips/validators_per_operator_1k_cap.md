|     Author     |               Title               | Category  |   Status    |    Date    |
| -------------- | --------------------------------- | --------- | ----------- | ---------- |
| Matheus Franco | 1K cap on validators per operator | Contracts | spec-merged | 2024-11-28 |

## Summary

This SIP proposes increasing the cap on the number of validators per operator from 500 to 1000.

## Motivation

The recent optimizations allowed operators to reduce their operational costs while running the same number of validators, thus keeping the same profits. Therefore, this allows operators to further increase the size of their validator sets, keeping a sustainable processing cost and increasing their profitability.

## Performance trade-offs

To understand the performance impact, we compared two settings with the same number of validators (namely, 104k), one with a cap of 500 and the other of 1000 validators per operator. Using the 500 case as a benchmark, we got the following results for the 1000 case:

| Metric                             | Impact     |
|------------------------------------|------------|
| Maximum operator cryptography cost | 20% higher |
| Average operator cryptography cost | 80% higher |
| Maximum operator message rate      | 30% lower  |
| Average operator message rate      | 22% lower  |

This behavior is expected. More validators per operator reduce the number of required operators and allow Committee Consensus to aggregate more duties.
This promotes a reduction in the number of exchanged messages and non-committee message ratios.
On the other hand, since the partial signature phase is the heaviest influence on the cryptography cost,
increasing the cap from 500 to 1k inevitably increases the cryptography cost.

The difference between 20% (average between operators) and 80% (maximum operator) in the cryptography cost increase is due to the effects of Committee Consensus.
Small and medium operators don't gain much from Committee Consensus, since the number of consensus instances increases linearly for a small number of validators.
On the other hand, for large operators, the number of consensus instances stays fixed at 32 per epoch (despite the increase of validators).

> [!NOTE]
> With the Pectra fork, dropping the `CommitteeIndex` field in the attestation beacon object will allow an operator to verify all post-consensus signatures in constant time. In other terms, currently, due to different `CommitteeIndex`s, the complexity of signatures verification on the post-consensus phase is $\mathcal{O}(d)$, where $d$ is the number of duties. After the Pectra fork, it will become $\mathcal{O}(1)$, significantly reducing the overall cryptography cost.

## Security

Post-consensus messages may increase their average and maximum sizes as shown in the [spec changes](#partialsignaturemessages). Nonetheless, it's still way smaller than a consensus proposal message. Thus, the proposed change doesn't introduce extra concerns on network security.

## Spec changes

### PartialSignatureMessages

The proposed change influences the number of signatures a post-consensus message may have.
Letting $v$ as the number of validators per operator, to account for attestation and sync committee duties, the maximum number of signatures per message is computed as

$$v + min(512, v)$$

Before we had $500 + min(512,500) = 1000$, now, we have $1000 + min(512,1000) = 1512$.

Thus, we need to change the `PartialSignatureMessages` structure in the following way:

```go
type PartialSignatureMessages struct {
    Type     PartialSigMsgType
    Slot     phase0.Slot
    Messages []*PartialSignatureMessage `ssz-max:"1512"` // Changed from 1000 to 1512
}
```

### Message Validation

In message validation, we need to update the following:
- for the rule on the number of signatures contained in a post-consensus message, the limit must be changed from 1000 to 1512.
- for the rules on the maximum byte size of a message, we have the new maximum sizes:
  - maximum encoded `PartialSignatureMessages`: 144020 $\to$ 217748
  - maximum encoded `SSVMessage` with encoded `PartialSignatureMessages`: 144088 $\to$ 217816
  - maximum encoded `SignedSSVMessage` with encoded `PartialSignatureMessages`: 147588 $\to$ 221316
