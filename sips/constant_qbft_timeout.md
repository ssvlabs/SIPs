| Author      | Title                 | Category | Status |
|-------------|-----------------------|----------|--------|
| Alon Muroch | Constant QBFT timeout | Core     | open-for-discussion  |

_Special thanks for Henrique Moniz for reviewing_

**Summary**  
Describes moving to a constant timeout for QBFT by defining better the network module for the SSV network.

**Rational**  
Timeouts in QBFT allow the protocol to overcome partially asynchronous network modules in which there is an unknown upper bound of latency. 
This unknown upper bound requires  the QBFT protocol to make progress in rounds, each round is considered a period of a synchronous network (known upper limit on latency).  
According to the [original QBFT paper](https://arxiv.org/pdf/2002.03613.pdf), timer t(r_i) "is an exponential function of the round number r_i..".

An exponential timeout (or even a sub-exponent) can quickly grow the timeout value under periods of low network stability (for example >f nodes are offline). In such cases recovery becomes a lengthy process, especially when the leader is faulty after a long period of instability.  

By making the timeout constant we can remove a lot of "stand and wait" post unstable periods.

**SSV Network Module**  
We adopt a slightly modified network module for SSV, it is still a partially synchronous module but with an added assumption that there are long enough periods of stability in which network latency is < T.  

From the above we can safely set the timeout to a constant T.

**Beacon Duty Timing Assumptions**  
The Beacon chain has some timing assumptions regarding duty execution for maximizing rewards.

| Duty           | Timing Assumption | Dependency     | Description                                                                                                         |
|----------------|-------------------|----------------|---------------------------------------------------------------------------------------------------------------------|
| Proposal       | < 4 Sec           | ---            | Attestations wait 4 seconds for a block                                                                             |
| Attester       | < 4 Sec           | Proposal       | Attestations wait 4 seconds for a block then need to be propogated within 4 seconds for aggregators to pick them up |
| Aggregator     | < 4 Sec           | Attester       | Aggregators start aggregating within 8 seconds from slot start and need to propogate within 4 seconds               |
| Sync Committee | ---               | ---            | ---                                                                                                                 |
| Aggregator     | ---               | Sync Committee | ---                                                                                                                 |


**Specification**  

New timeout calculation:

```go
// RoundTimeout returns the number of seconds until next timeout for a give round
func (i *Instance) RoundTimeout(round Round) uint64 {
	return 2
}
```

We set timeout to 2 seconds as it's enough for any duty to be executed fully but also leaves a "second chance" in case a leader is faulty.