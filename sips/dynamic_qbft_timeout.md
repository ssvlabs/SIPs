| Author         | Title                 | Category | Status   |
| -------------- | --------------------- | -------- | -------- |
| Gal Rogozinski |  QBFT timeout | Core     | approved |


**Summary**  
Changes to the constant timeouts we defined in SIP#6 that should improve performance. 

**Rational**  
Currently for all duties, we have a `quickTimeout` of 2 seconds before sending a `RoundChange` message and a `slowTimeout` of 2 minutes starting from the 8th round.The timer starts when the QBFT instance starts. This approach has 2 issues:

1. Duties that require a pre-consensus stage may start their instances at different times, causing some nodes to timeout sooner than others. This will result in a greater probability of a failed first round.
2. Different duties have different timing assumptions, which may cause some duties to execute earlier than others.



**Beacon Duty Timing Assumptions**  
The Beacon chain has some timing assumptions regarding duty execution for maximizing rewards.

| Duty           | Timing Assumption | Dependency     | Description                                                                                                         |
| -------------- | ----------------- | -------------- | ------------------------------------------------------------------------------------------------------------------- |
| Proposal       | < 4 Sec           | ---            | Attestations wait 4 seconds for a block                                                                             |
| Attester       | < 4 Sec           | Proposal       | Attestations wait 4 seconds for a block then need to be propogated within 4 seconds for aggregators to pick them up |
| Aggregator     | < 4 Sec           | Attester       | Aggregators start aggregating within 8 seconds from slot start and need to propogate within 4 seconds               |
| Sync Committee | < 4 sec           | Proposal       | Sync Committee wait 4 seconds for a block                                                                           |
| Contribution   | < 4 sec           | Sync Committee | Contributions wait 8 seconds from the slot start                                                                    |

        

**Specification** 

```go
var (
    quickTimeoutThreshold = Round(8)
    quickTimeout          = 2 * time.Second
    slowTimeout           = 2 * time.Minute
)


// Timeout returns the number of seconds until next timeout for a given round.
// When we have the proposal duty we immediately call it.
func Timeout(r Round) time.Duration {
    if r <= quickTimeoutThreshold {
        return quickTimeout
    }
    return slowTimeout
}

// Called for all beacon duties other than proposals
func RoundTimeout(r Round, baseDuration Duration) Time {
    if r == Round.FirstRound {
        return baseDuration + quickTimeout
    } else {
        return Timeout(r)
    }
}
```

*Proposals*

No changes from SIP#6.

```go
func ProposalTimeout(r Round)
    return Timeout(r)
```

*Attestations/Sync Committees*

Once the slot starts, wait 4 seconds for a block to be produced then start the timer according to the same rules as defined in SIP#6.

```go
func AttestationOrSyncCommitteeTimeout(r Round)
    return RoundTimeout(r, 4)
```

*Aggregator/Contribution*

Once the slot starts, wait 8 seconds for a block to be produced then start the timer according to the same rules as defined in SIP#6.

```go
func AggregationOrContribuitionTimeout(r Round)
    return RoundTimeout(r, 4)
```