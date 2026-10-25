| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Gal Rogozinski ([@GalRogozinski](https://github.com/GalRogozinski)) | Non-Proposer QBFT Round Timers for Glamsterdam | Core | draft | [SIP-6](./constant_qbft_timeout.md), ePBS Support ([#94](https://github.com/ssvlabs/SIPs/pull/94)) | 2026-10-25 |

**Summary**

Every QBFT role except the proposer runs a slot-anchored round timer: round `r` expires at `slot_start + head_start(role) + quickTimeout × r`. The head start is one or two thirds of the slot, which were the attestation and aggregate deadlines before Glamsterdam. Glamsterdam (ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)) moves those deadlines to 25% and 50% of the slot. If the head starts stay where they are, a committee whose round-1 leader is faulty starts round 2 at 6s, which is the new aggregate deadline, instead of 2s ahead of it.

This SIP pins each slot-anchored role's head start to that role's beacon deadline for the slot's fork. From the Gloas fork on, the committee head start is 3s and the aggregator, sync-committee-contribution and aggregator-committee head starts are 6s. The quick timeout (2s), the quick-timeout threshold, the slow timeout and the per-role round caps are unchanged. The proposer timer is covered by SIP-102 ([#102](https://github.com/ssvlabs/SIPs/pull/102)) and is out of scope.

Tracking issue: [#104](https://github.com/ssvlabs/SIPs/issues/104).

**Rational & Design Goals**

*Slot timing under Glamsterdam.* From the `gloas` consensus specs ([`specs/gloas/validator.md`](https://github.com/ethereum/consensus-specs/blob/dev/specs/gloas/validator.md)), with a 12s slot:

| Deadline | Pre-Gloas | Gloas | Spec constant | Roles that wait for it |
| -------- | --------: | ----: | ------------- | ---------------------- |
| Attestation due | 4s | **3s** | `ATTESTATION_DUE_BPS_GLOAS = 2500` | committee |
| Sync committee message due | 4s | **3s** | `SYNC_MESSAGE_DUE_BPS_GLOAS = 2500` | committee |
| Aggregate due | 8s | **6s** | `AGGREGATE_DUE_BPS_GLOAS = 5000` | aggregator, aggregator-committee |
| Sync contribution due | 8s | **6s** | `CONTRIBUTION_DUE_BPS_GLOAS = 5000` | sync-committee contribution, aggregator-committee |

*What the head start is for.* A slot-anchored instance does not start at a fixed time. A committee instance starts when the slot's block arrives (plus a short propagation delay), or at the attestation deadline if no block has arrived by then. An aggregator-type instance fetches its value at the aggregate deadline. So the role's deadline is the latest time its instance normally starts, and the head start guarantees that round 1 always gets one full `quickTimeout` after that point. That property is the one this SIP keeps.

*Why the old head starts break under Gloas.* The table shows the time-into-slot at which each round ends.

| Role | Round 1 ends | Round 2 ends | Next deadline the duty feeds |
| ---- | -----------: | -----------: | ---------------------------- |
| Committee, pre-Gloas | 6s | 8s | aggregation at 8s |
| Committee, Gloas, old head start (4s) | 6s | 8s | aggregation at **6s** |
| **Committee, Gloas, this SIP (3s)** | **5s** | **7s** | aggregation at 6s |
| Aggregator-type, pre-Gloas | 10s | 12s | next slot's proposal at 12s |
| Aggregator-type, Gloas, old head start (8s) | 10s | 12s | next slot's proposal at 12s |
| **Aggregator-type, Gloas, this SIP (6s)** | **8s** | **10s** | next slot's proposal at 12s |

Committee: today a round change hands round 2 to a new leader at 6s, 2s before aggregators collect. Under Gloas with the old head start, round 2 begins exactly when aggregators collect, so a single faulty leader pushes every attestation in the committee past the aggregation window. With a 3s head start, round 2 begins at 5s, 1s before aggregation. The attestation still lands late for the head vote if round 2 is needed, but it reaches aggregators in the same slot instead of the next one.

Aggregator-type roles: the old head start stays correct in shape, but it waits 2s after the value is already available. The instance cannot start before the aggregate deadline, so those 2s are pure delay. Re-pinning restores the 4s of round-2 slack before the next slot's proposal that these roles have today.

*Design goals, in priority order:*

1. Round 1 MUST get at least one full `quickTimeout` after the role's latest normal instance start. A shorter window would force a round change on every duty whose instance starts at the deadline fallback.
2. Round 2 MUST start no later than the next downstream deadline: before aggregation for the committee, and with the same slack before the next slot's proposal for aggregator-type roles as before Gloas.
3. The timer MUST stay a pure function of `(role, slot, round)` and the beacon config, so that message validation can keep estimating a peer's round from the clock.
4. No change to message formats, signing, domains, round caps or the quick, slow or threshold constants.

**Specification**

*Definitions*

- `slot_start(slot)`: wall-clock time, beacon genesis plus `slot × SECONDS_PER_SLOT`. `slot` is the QBFT message's `Height`.
- `is_gloas(slot)`: `compute_epoch_at_slot(slot) >= GLOAS_FORK_EPOCH`. The selection below MUST use the duty's slot, never the current wall-clock slot, so a late instance for a pre-fork slot keeps pre-fork timing.
- `interval(slot)`: `SECONDS_PER_SLOT / 4` if `is_gloas(slot)`, otherwise `SECONDS_PER_SLOT / 3`. On mainnet this is 3s or 4s. It equals the attestation and sync-message deadline of the slot's fork, and `2 × interval(slot)` equals the aggregate and contribution deadline.

*Head start*

```go
// headStart returns the time into the slot after which round 1 runs one quickTimeout.
// It equals the beacon deadline the role waits for, in the slot's own fork.
func headStart(role RunnerRole, slot Slot) time.Duration {
    switch role {
    case RoleCommittee:
        return interval(slot) // attestation and sync-message due
    case RoleAggregator, RoleSyncCommitteeContribution, RoleAggregatorCommittee:
        return 2 * interval(slot) // aggregate and contribution due
    default:
        return 0 // round-relative roles (proposer); see SIP-102
    }
}
```

*Round expiry* (unchanged formula, new head start)

```go
var (
    quickTimeoutThreshold = Round(8)        // unchanged (SIP-6)
    quickTimeout          = 2 * time.Second // unchanged (SIP-6) for every slot-anchored role
    slowTimeout           = 2 * time.Minute // unchanged (SIP-6)
)

// roundExpiry returns the time into the slot at which round r ends for a slot-anchored role.
func roundExpiry(role RunnerRole, slot Slot, r Round) time.Duration {
    if r <= quickTimeoutThreshold {
        return headStart(role, slot) + time.Duration(r)*quickTimeout
    }
    return headStart(role, slot) +
        time.Duration(quickTimeoutThreshold)*quickTimeout +
        time.Duration(r-quickTimeoutThreshold)*slowTimeout
}
```

- An operator running a slot-anchored instance MUST, on entering round `r`, arm a timer that fires at `slot_start(slot) + roundExpiry(role, slot, r)`. If that instant has already passed, the timer fires immediately. The existing `UponRoundTimeout` behaviour on expiry is unchanged.
- Per-role round caps are unchanged: 12 for committee, aggregator and aggregator-committee, 6 for sync-committee contribution.
- Message validation MUST estimate a peer's current round for slot-anchored roles by inverting `roundExpiry` with the same `headStart(role, slot)`. The round-spread tolerances (`allowedRoundsInPast = 2`, `allowedRoundsInFuture = 1`) are unchanged.

*Expected values (mainnet, 12s slot)*

| Role | `headStart`, pre-Gloas slot | `headStart`, Gloas slot | Round 1 ends (Gloas) | Last round ends (Gloas) |
| ---- | --------------------------: | ----------------------: | -------------------: | ----------------------: |
| Committee | 4s | 3s | 5s | round 12: 3s + 16s + 4 × 120s = 499s |
| Aggregator | 8s | 6s | 8s | round 12: 502s |
| Aggregator-committee | 8s | 6s | 8s | round 12: 502s |
| Sync-committee contribution | 8s | 6s | 8s | round 6: 18s |
| Proposer | n/a (round-relative) | n/a (round-relative) | SIP-102 | SIP-102 |

Conformance cases an implementation SHOULD test: each row above for both forks; the last pre-fork slot and the first Gloas slot; a timer armed for a pre-fork slot after the fork epoch has begun, which MUST use the pre-fork head start; and round estimation at each round boundary agreeing with `roundExpiry`.

*Compatibility and rollout*

- No message, signature or domain change.
- The change activates at `GLOAS_FORK_EPOCH` with the rest of Glamsterdam support. It needs no separate fork: the head start is selected by the duty's slot, so every Gloas-capable operator switches at the same slot. An operator without Gloas support cannot run Gloas duties anyway.
- ssv already implements this behaviour (`IntervalDuration` in `networkconfig/beacon.go` and `round1HeadStart` in `protocol/v2/qbft/roundtimer/timer.go`, shipped with ePBS support). Anchor MUST match it before the fork. ssv-spec defines only the timeout constants, not the slot-anchored schedule, so it needs no change.

**Security Considerations**

- *Consensus safety.* QBFT safety does not depend on timeout values. Round-change justification and message-validation semantics are unchanged. No impact.
- *Liveness.* Round 1 keeps a full `quickTimeout` after the latest normal instance start, so an honest leader has the same time to decide as today. The only change is that every round of a Gloas slot ends 1s (committee) or 2s (aggregator-type) earlier relative to slot start, which tracks the 1s and 2s earlier instance starts. An instance that starts on a block arriving before the new deadline gets a longer round 1 than its start-to-deadline gap, exactly as today.
- *Message validation.* Round estimation moves with the head start. If one client applied the Gloas head start and another did not, their estimates for the same message would differ by up to one round: 1s against a 2s round for the committee, and exactly 2s, one full round, for aggregator-type roles. The `allowedRoundsInFuture = 1` slack absorbs the committee case and only barely absorbs the aggregator case, with no room for clock drift. Clients that disagree on this would reject each other's valid messages and down-score peers. This is why both clients MUST select the head start by the duty's slot fork, and why the change ships with Gloas support rather than ahead of it.
- *Slashing.* Unchanged. Earlier round changes do not create a new way to sign two values: an instance still decides at most one value, and the slashing-protection checks before signing are untouched.
- *Fork boundary.* An instance for the last pre-fork slot can still be running after the fork epoch starts. Because `is_gloas` is evaluated on the duty's slot, it keeps the pre-fork schedule to the end, and both sender and validator agree on it.

**Alternatives Considered** (non-normative)

1. *Keep the pre-Gloas head starts.* The committee's round 2 begins at the 6s aggregate deadline, so a faulty round-1 leader costs the whole committee's attestations a slot of aggregation. Aggregator-type instances wait 2s after their value is available for no benefit.
2. *Shorten `quickTimeout` instead of the head start.* A 1s committee `quickTimeout` with a 4s head start ends round 1 at 5s too, but a stalled instance would then cycle through quick rounds twice as fast, and a deadline-fallback start would get only 1s for round 1. It also changes later rounds' timing, which this problem does not require.
3. *End round 1 at the deadline (head start = deadline − `quickTimeout`).* An instance that starts at the deadline fallback would time out round 1 immediately, forcing a round change on every duty whose block is late.
4. *Instance-relative timers, as for the proposer.* This removes clock-based round estimation, which message validation relies on for these roles. It would need a new validation rule and is a larger change than the deadline shift calls for.

**Open Questions**

- Whether 2s is still the right `quickTimeout` for the committee under a 3s attestation deadline is not settled here. This SIP preserves today's round-1 budget and changes only where it is anchored. Revisiting it needs Gloas-era timing data (block arrival, round-1 duration) that only exists after the fork.
