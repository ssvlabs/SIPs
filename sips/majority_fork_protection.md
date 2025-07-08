|     Author     |           Title           | Category |       Status        |    Date    |
| -------------- | ------------------------- | -------- | ------------------- | ---------- |
| Alan Li        | Majority Fork Protection | Core     | open-for-discussion  | 2025-07-08 |

**Summary**  
This SIP addresses a slashing risk in SSV Network DVT that arises during client majority forks, as observed in the Holesky testnet [incident](https://etherworld.co/2025/02/24/holesky-testnet-incident-postmortem-report/) after the Pectra upgrade. When a majority execution client (e.g., Geth) develops a bug and forks from the canonical chain, DVT validators can be pulled onto the faulty fork, risking slashing if they attempt to return to the correct chain. This SIP proposes a validation-based solution to improve validator safety by ensuring that only attestations matching an operator's own view are signed.

**Rational & Design Goals**  
The goal is to enhance validator safety and network resilience by preventing DVT validators from being slashed due to majority client bugs. The design aims to:
- Ensure DVT validators only sign attestations with source and target epochs agreed upon by a quorum of operators.
- Maintain compatibility with asynchronized clients without introducing unnecessary complexity.

**Specification**  

## Problem
In Ethereum, consensus relies on validators attesting to blocks according to justified checkpoints. When an execution client develops a bug that causes it to fork from the canonical chain, validators using that client may unknowingly attest to an invalid chain.

Consider 1 majority client and 3 minority clients:
- **Majority**: Geth (produces an invalid block due to a bug)
- **Minority 1**: Besu
- **Minority 2**: Erigon
- **Minority 3**: Nethermind

In a **solo validator** setup, if the validator's client is Geth, it goes onto the faulty fork. But if the validator is using Besu or Erigon or Nethermind client, it remains safe.

In contrast, in an **SSV DVT setup**, the validator relies on a committee of multiple operators, each running potentially different clients. If even one operator runs a faulty client, the validator can be pulled onto the faulty fork. When the validator tries to switch away from the faulty chain it risks slashing as we will see below.

### Attestation Slashing Rule
In the DVT setup, when a majority-client operator proposes an attestation from a faulty fork, other minority operators will sign as long as the proposal is not slashable given what they signed before. 

In other words, an operator must not sign two attestations `data1` and `data2` that satisfy any of the following rules:

**Double vote rule**: 
`data1 != data2 and data1.target == data2.target`

Example: 
```
data1.source = 0, data2.source = 1
data1.target = 2, data2.target = 2
```

**Surrounding rule**: 
`data1.source < data2.source and data1.target > data2.target`

Example: 
```
data1.source = 0, data2.source = 1
data1.target = 3, data2.target = 2
```

### DVT Example with 1-out-of-4 Majority Client

![Example](https://github.com/user-attachments/assets/8fa93da1-f76f-4e03-9e7d-cb91f89d3cf2)
*A fork with faulty majority on the left side and correct minority on the right side. The numbers denote the epochs. (0,1) represents an attestation with source epoch 0 and target epoch 1. Dotted arrows between the chains represent proposals of new attestation data when advancing epochs in the DVT setting.*

#### The global view
Let's say at epoch 1 faulty majority creates an incorrect block, here a fork is created. At this point, the attestation sources and targets for both sides are still the same. 

Starting from epoch 2, the attestations are different. The majority client attests (1,2) because on this side 0 is finalized and 1 is justified as they are the majority. The minority clients attest (0,2) because 1' is not justified as they are the minority.

Similarly, in the following epoch we have (2,3) for the majority and (0,3) for the minority, and so on.

#### The DVT view
Still consider a DVT validator with 4 operators running 4 different clients, 1 faulty majority and 3 correct minorities.

At epoch 1 a DVT validator attests (0,1) in all circumstances. 

Starting from epoch 2 it depends on who is the attestation proposer of the epoch. 

- **If majority leads**, the attestation proposal is (1,2). 
    - Minority clients agree because signing (1,2) is not slashable given (0,1) was signed. 
    - Because the minority-proposed attestation (0,3) is now slashable due to the surrounding rule (having already signed (1,2)), the validator cannot safely return to the minority chain.
- **If minority leads**, the attestation proposal is (0,2). 
    - The majority client agree because signing (0,2) given (0,1) was signed is not slashable. 
    - However, in the following epoch, proposals from the faulty majority can be signed because signing (2,3) given (0,2) was signed is not slashable.

To conclude, if the majority is signed there is no way back to minority. Also because the majority client operator will eventually propose attestation due to the round-robin proposer selection mechanism, the validator will eventually only sign faulty majority attestation, even though there are 3 correct minority operators.

## Solution
The proposed solution is to simply add a validation rule before signing attestation. A client needs to get its attestation data from the beacon. Then the client only signs a proposed attestation if the source and target match its own attestation data. 

```go
func shouldSignAttestation(own, proposed Attestation) bool {
    return own.Source == proposed.Source && own.Target == proposed.Target
}
```

**As a result, a DVT validator will only sign the attestations from the chain that the operators have quorum.**

### Incentive Problem when No Quorum
Suppose a DVT validator has no quorum on both the majority and minority sides. In the above example, it seems to be a better decision to keep signing the minority instead of signing nothing because it can get more rewards (since minority is safer, because it can move to the majority). 

However, since during a fork we don't know which branch is correct, simply signing the safer side is still not a good approach. This is under the consideration that the proposer can report fake forks and propose random attestation data. If such a rule exists a malicious validator can always use it to reduce validator rewards. Even if the other fork is detected as the correct fork, the client should stick to its own fork choice algorithm that picks the heavier valid fork. Therefore, when there is no quorum, no attestations should be signed.

**Drawback**: when no quorum attestations will not be signed, it is safe but no rewards until social consensus is reached. We are also hoping the fork will be resolved soon, otherwise the validator will incur penalties for being inactive.

### Asynchronized Clients
For different clients are in the same fork, we consider the edge cases that they can have different attestations at a point of time due to asynchronization, and we conclude that asynchronization does not affect our majority fork protection mechanism.

Consider a client _A_ with full, up-to-date information and a client _B_ in the same fork that is lagged or has missing information. There are following possibilities regarding asynchronization of attestations:
1. **Target epochs are different**: _A_ enters the first slot of epoch 2 while _B_ is still at the end of epoch 1. They will have attestations with different sources and targets. This scenario is fine because _B_ will not participate in the committee, so _B_ will not vote for the attestation at all. 
2. **Block roots are different**: _A_ and _B_ both enter epoch 2 and receive the attestation duty, but _A_ has received the block proposal in the first slot of epoch 2, while _B_ has not received this block proposal. This gives the same attestation sources and targets for _A_ and _B_, but they will have different target block roots. In this case _B_ can sign _A_'s attestation and vice versa. Attestations are not slashable with invalid block root, so the worst case is missing rewards. 
3. **Source epochs are different**: Source epochs can be different due to asynchronization, for example, _B_ has missing attestations data in the previous epoch. In this case, looking at the attestation proposed by _A_, it is indistinguishable whether _B_ is missing attestation for the source epoch or if _A_ and _B_ are in different chains. In this case _B_ should simply not sign the attestation, suggested by our majority fork protection rule. 
Finding out whether _B_ is in the same fork but has missing attestation data in a previous epoch is not efficient, _A_ needs to send over all relevant attestation data for _B_ to verify. Given this case is not so common, it does not result in slashing, and the verification cost is high, it is better not to handle this case with additional care.

## Properties
The majority fork protection mechanism guarantees that: 

A DVT validator only signs attestations with source and target epochs agreed by a quorum of operators.

Consider a DVT validator with $n=3f+1$ operators, where $f$ is the maximum number of Byzantine fault operators it can tolerate. Each of these $n$ operators can have different target and source epochs, due to forking or asynchronization. The properties we get based on our attestation validation mechanism are:

1. Only with at least $2f+1$ operators sharing the same source and target epochs, attestations will be signed.
2. When no quorum, no attestations will be signed.

## Conclusion
With the attestation validation mechanism, DVT enhances the network's resilience against bugs in majority clients. By leveraging a diverse set of clients across operators, validators are more likely to stay on the correct chain even when the majority fork is faulty. From a validator's perspective, this significantly reduces the risk of slashing. From the network's perspective, if over 33% of validators adopt DVT, such forks can be effectively mitigated, preventing them from escalating into systemic issues.