| Author         | Title                 | Category | Status   |
| -------------- | --------------------- | -------- | -------- |
| Gal Rogozinski |  QBFT timeout | Core     | approved |


**Summary**  
Changes to the constant timeouts we defined in [SIP#6](https://github.com/bloxapp/SIPs/blob/main/sips/constant_qbft_timeout.md) that should improve performance. 

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

From the slot start time after `baseDuration` delay wait 2 seconds before emitting a `RoundChange` message until the 8th round (inclusive). Afterwards, wait 2 minutes.

```go
var (
    quickTimeoutThreshold = Round(8)
    quickTimeout          = 2 * time.Second
    slowTimeout           = 2 * time.Minute
)

// RoundTimeout returns the time in which we should send a RC message
// Called for all beacon duties other than proposals
func RoundTimeout(r Round, dutyStartTime Time, baseDuration Duration) Time {
   // Calculate additional timeout based on round
	var additionalTimeout time.Duration
	if round <= quickTimeoutThreshold {
		additionalTimeout = time.Duration(int(round)) * slowTimeout
	} else {
		quickPortion := time.Duration(quickTimeoutThreshold) * quickTimeout
		slowPortion := time.Duration(int(round-quickTimeoutThreshold)) * slowTimeout
		additionalTimeout = quickPortion + slowPortion
	}

	// Combine base duration and additional timeout
	timeoutDuration := baseDuration + additionalTimeout

	// Get the start time of the duty
	dutyStartTime := t.beaconNetwork.GetSlotStartTime(phase0.Slot(height))

	// Calculate the time until the duty should start plus the timeout duration
	return time.Until(dutyStartTime.Add(timeoutDuration))
}
```

*Proposals*

No changes from [SIP#6](https://github.com/bloxapp/SIPs/blob/main/sips/constant_qbft_timeout.md). Doesn't rely on the slot/duty start time.

*Attestations/Sync Committees*

Once the slot starts, wait 4 seconds for a block to be produced then start the timer according to the new rules.

```go
func AttestationOrSyncCommitteeTimeout(r Round, dutyStartTime Time) Time
    return RoundTimeout(r, dutyStarTime, 4)
```

*Aggregator/Contribution*

Once the slot starts, wait 8 seconds for a block to be produced then start the timer according to the new rules

```go
func AggregationOrContribuitionTimeout(r Round, dutyStartTime Time) Time
    return RoundTimeout(r, dutyStartTime 8)
```