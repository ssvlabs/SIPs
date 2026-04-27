|     Author     | Title                            | Category |       Status        | Date       |
| -------------- |----------------------------------| -------- | ------------------- |------------|
| Matheus Franco | Epoch-Aware Round-Robin Proposer | Core     | spec-merged         | 2025-12-13 |


## Summary

Include the duty's Ethereum epoch in the Round-Robin rotation function for determining the QBFT proposer. 

## Motivation

The previous Round-Robin function took into account only the duty's slot and the QBFT round number.
Because there are 32 slots in an epoch and 32%4=0,
committees with 4 operators had the same profile of proposers every epoch.
I.e., the i-th slot of every epoch had always the same first proposer.

## Rationale

Since the current function already takes the duty's slot into account
(given by `state.Height`),
we don't even need to change the function signature.
We can simply compute the current epoch through $\lfloor \frac{Height}{SLOTS\_PER\_EPOCH} \rfloor$
and use it in the rotation computation.

```go
type ProposerF func(state *State, round Round) types.OperatorID
```

## Improvements

The new function accomplishes a more "unpredictable" rotation.
Even though it doesn't truly add randomness,
it removes an easily exploitable, repeating schedule.

In the future, we could turn the function into a pseudo-random one,
making it harder to predict the proposer,
though maintaining the Round-Robin fairness property.

## Spec change

The function signature is kept unchanged, and the
body is changed in the following way:
```go
func RoundRobinProposer(state *State, round Round) types.OperatorID {
	firstRoundIndex := 0
	if state.Height != FirstHeight {
		firstRoundIndex += int(state.Height) % len(state.CommitteeMember.Committee)
	}
	ethEpoch := int(state.Height) / SLOTS_PER_EPOCH // NEW: compute the current Ethereum epoch

	index := (firstRoundIndex +
		int(round) - int(FirstRound) +
		ethEpoch) // NEW: include the epoch in the rotation
		% len(state.CommitteeMember.Committee)
	return state.CommitteeMember.Committee[index].OperatorID
}
```
