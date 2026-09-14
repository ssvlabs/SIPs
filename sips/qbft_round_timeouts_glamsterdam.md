| Author | Title | Category | Status | Date |
| ------ | ----- | -------- | ------ | ---- |
| Gal Rogozinski ([@GalRogozinski](https://github.com/GalRogozinski)) | QBFT Round Timeouts for Glamsterdam | Core | draft | 2026-09-06 |

**Summary**

Glamsterdam (ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)) moves the beacon attestation deadline from 4s to 3s into the slot. The proposer duty's QBFT round timeout is 2s from instance start ([SIP-6](./constant_qbft_timeout.md)), and a proposer instance starts only after RANDAO pre-consensus and block retrieval, typically 1.1–1.5s into the slot. Under the new deadline a round change therefore starts round 2 after attesters have already voted: a round change (~0.18% of proposer duties) goes from usually recoverable to usually fatal, roughly tripling missed proposals (0.056% → 0.166%).

This SIP lowers the proposer duty's quick round timeout from 2000ms to 1500ms, keeps the timer anchored at instance start, and makes the message-validation round estimate consistent with the new value. Round timeouts for the other duties under Glamsterdam slot timing are listed as **TBD** and will be specified in a revision of this SIP.

**Rational & Design Goals**

*Slot timing under Glamsterdam.* From the `gloas` consensus specs ([`specs/gloas/validator.md`](https://github.com/ethereum/consensus-specs/blob/dev/specs/gloas/validator.md)), durations are expressed in basis points of the 12s slot. Constant names are verbatim from that file: the `_GLOAS` suffix marks constants that override an existing phase0 value, while `PAYLOAD_DUE_BPS` and `PAYLOAD_ATTESTATION_DUE_BPS` are new in gloas and have nothing to suffix. The "Today" column is derived as `SECONDS_PER_SLOT // INTERVALS_PER_SLOT`, not from bps constants, so the two columns are different kinds of value:

| Event | Today | Glamsterdam | Spec constant |
| ----- | ----: | ----------: | ------------- |
| Beacon block proposal | 0s | 0s | — |
| Attestation due | 4s | **3s** | `ATTESTATION_DUE_BPS_GLOAS = 2500` |
| Sync committee message due | 4s | **3s** | `SYNC_MESSAGE_DUE_BPS_GLOAS = 2500` |
| Aggregate due | 8s | **6s** | `AGGREGATE_DUE_BPS_GLOAS = 5000` |
| Sync contribution due | 8s | **6s** | `CONTRIBUTION_DUE_BPS_GLOAS = 5000` |
| Execution payload due | — | 6s | `PAYLOAD_DUE_BPS = 5000` |
| Payload attestation due | — | 9s | `PAYLOAD_ATTESTATION_DUE_BPS = 7500` |

The 6s slot proposal (EIP-7782) was declined for Glamsterdam; this SIP assumes a 12s slot.

*Why the proposer timeout is the urgent part.* A proposer QBFT instance starts at `slot_start + RANDAO + ProposerDelay + block_fetch`. Measured on mainnet (30 days, 25,126 proposer duties, [ssv#3018](https://github.com/ssvlabs/ssv/pull/3018)):

| Population | Instance start (round-1 proposal) |
| ---------- | --------------------------------: |
| SSV Labs operators, `ProposerDelay = 0` | 1.1s median |
| Network | 1.1–1.5s typical (median decision 1.44s) |
| Fastest clusters (local block building) | 0.5–0.9s |
| Clusters with `ProposerDelay ≈ 0.9s` | ~2.0s |

With a 2s round-1 timeout, round 2 begins at `start + 2s`, i.e. after 3.1s for a typical cluster. Today a round-2 block deciding in the 3.0–3.5s band still lands ~98% of the time (1.9% miss rate, Appendix A); after Glamsterdam it lands never. Projected effect, same duty distribution, deadline moved to 3s (model calibrated to reproduce today's 14 misses):

| Scenario | Missed proposals / month | Share of proposer duties | Round-change duties missed (of ~45) |
| -------- | -----------------------: | -----------------------: | ----------------------------------: |
| Today (4s deadline, 2000ms) | 14 | 0.056% | 9 (20%) |
| Glamsterdam, 2000ms | ~42 | 0.166% | ~33 (74%) |
| **Glamsterdam, 1500ms** | **~26** | **0.105%** | **~18 (39%)** |
| Glamsterdam, 1250ms | ~23 | 0.091% | — |
| Glamsterdam, 1000ms | ~22 | 0.087% | — |

The 1250ms and 1000ms rows are within the 95% interval of the 1500ms row (22–32/month). Round 1, when it succeeds, is fast: median 59ms, p99 191ms, slowest success in 30 days 1,148ms. A 1500ms timer never fires on a round 1 that was going to succeed; a 1000ms timer would fire on 5–12 real duties a month.

*Design goals, in priority order:*

1. A round change MUST leave round 2 enough time to decide and broadcast before the 3s attestation deadline for clusters that start consensus by ~1.3s.
2. The timer MUST NOT fire on a round 1 that would have succeeded: the timeout stays above the observed maximum round-1 duration with margin.
3. The round-1 budget MUST stay independent of `ProposerDelay`, so the timer remains anchored at instance start, not slot start. A slot-anchored deadline short enough to help typical clusters would fire before round 1 even begins for clusters using a deliberate delay.
4. No change to message formats, signing, or domains; pre-upgrade and post-upgrade operators MUST interoperate within one cluster.

**Specification**

*Definitions*

- `slot_start(height)`: beacon genesis time plus `height × SECONDS_PER_SLOT`.
- `round_entered_at(round)`: the operator's local time when its instance entered `round`. For `round = 1` this is the instance start; for `round > 1` it is the moment the instance advanced to that round (on timeout or on receiving `f+1` round-change messages for it).
- `round_timeout(role, round)`: the duration after `round_entered_at(round)` at which the operator MUST broadcast a `RoundChange` for `round + 1` if it is still in `round` (the existing `UponRoundTimeout` behaviour; unchanged).

*Proposer duty*

```go
var (
    quickTimeoutThreshold = Round(8)            // unchanged (SIP-6)
    quickTimeout          = 2 * time.Second     // unchanged (SIP-6)
    proposerQuickTimeout  = 1500 * time.Millisecond // new: was quickTimeout
    slowTimeout           = 2 * time.Minute     // unchanged (SIP-6)
)

// RoundTimeout returns the duration until the next timeout for a given role and
// round. The timer is armed on entering a round and is relative to that entry,
// not to slot start.
func RoundTimeout(role types.RunnerRole, r Round) time.Duration {
    if r > quickTimeoutThreshold {
        return slowTimeout
    }
    if role == types.RoleProposer {
        return proposerQuickTimeout
    }
    return quickTimeout
}
```

This amends [SIP-6](./constant_qbft_timeout.md)'s `RoundTimeout`, which took only a round and returned `quickTimeout` for every role.

- An operator running a proposer instance MUST arm a timer of `RoundTimeout(RoleProposer, round)` upon entering each round and MUST broadcast `RoundChange(round + 1)` on expiry if it is still in `round`.
- The proposer timer MUST be anchored at `round_entered_at`, never at `slot_start`.

*Other duties* — **TBD**

Glamsterdam also moves the attestation, sync-message, aggregate and contribution deadlines. Their timers are slot-anchored in the current node implementation and MAY need new base offsets. Values below are placeholders for review and MUST be filled before this SIP leaves `draft`.

| Duty | Current timer (node) | Glamsterdam deadline | Proposed | Status |
| ---- | -------------------- | -------------------- | -------- | ------ |
| Committee (attester + sync committee message) | `slot_start + 4s + 2s × round` for `round ≤ 8` | attestation 3s, aggregation starts 6s | TBD | TBD |
| Aggregator | `slot_start + 8s + 2s × round` for `round ≤ 8` | aggregate due 6s | TBD | TBD |
| Sync committee contribution | `slot_start + 8s + 2s × round` for `round ≤ 8` | contribution due 6s | TBD | TBD |
| Validator registration, voluntary exit | no QBFT | — | not applicable | — |

*Compatibility and rollout*

- No message, signature or domain change. Pre-upgrade and post-upgrade operators MAY coexist in one cluster.
- Round-change quorum in a mixed cluster: upgraded operators broadcast `RoundChange(2)` at `start + 1.5s`, non-upgraded at `start + 2s`. Once `f + 1` operators in a cluster are upgraded, the existing partial-quorum rule advances the rest at 1.5s; before that the cluster behaves as today. The change is never worse than the status quo during rollout.
- Pre-upgrade validators estimating rounds with 2000ms accept every post-upgrade proposer message for rounds 1–5. A round-6 message is dropped by them only if the instance started before 0.5s into the slot; a round-6 proposal cannot produce a canonical block in any case.
- The change MUST be active on a majority of mainnet operators before the Glamsterdam activation epoch. Fork assignment: **TBD** (see `forks/`).

**Security Considerations**

- *Consensus safety.* QBFT safety does not depend on timeout values; round-change justification and message validation semantics are unchanged. No impact.
- *Liveness.* A shorter round-1 window tolerates a slower honest leader less. The bound is empirical: 350ms above the slowest round-1 success observed in 30 days across 24,984 round-1 decisions, with simulated newly induced round changes of ~1–2 per month network-wide. Round 2 remains available. Under mixed versions liveness is bounded below by today's behaviour.
- *Slashing.* Unchanged. An instance decides at most one value and the block is signed only after the decision; a round change never yields a second signed block for the same slot.
- *Message validation surface.* For proposer messages the allowed round advances 25% faster with time, so a validator accepts marginally more distinct rounds per second; the cap of round 6 and the existing per-round duplicate limits are unchanged. Negligible.
- *Operator incentives.* The instance-start anchor keeps `ProposerDelay` an operator choice. Under a 3s deadline a delayed cluster that round-changes misses regardless of this SIP (`~2.0s + 1.5s > 3s`); operators SHOULD keep `ProposerDelay` at or below the existing 1s cap.

**Alternatives Considered** (non-normative)

1. *Slot-anchored proposer timer* ([ssv#2429](https://github.com/ssvlabs/ssv/issues/2429), the approach of [SIPs#37](https://github.com/ssvlabs/SIPs/pull/37) for other duties). To help typical clusters under a 3s deadline the round-1 deadline would sit at ~1.5–2.0s from slot start, which is before or at round-1 start for clusters using `ProposerDelay ≥ 0.5s`, forcing a round change on every such duty. Deferred; worth revisiting if ePBS bid retrieval shortens consensus start enough to remove the case for `ProposerDelay`.
2. *1250ms or 1000ms.* Projected gain over 1500ms is inside the error bars; 1000ms would time out 5–12 real round-1s per month.
3. *No change.* Missed proposals roughly triple (0.056% → 0.166%).

**Open Questions**

- Should the proposer instance stop at a lower round (`CutoffRound`, `MAX_ROUND`) given no round after 2 can land a block under a 3s deadline?
- ePBS replaces relay payload fetch with builder-bid selection; if consensus start drops well below 1s the slot-anchored alternative becomes attractive. Re-measure after the fork.
- Fill in the TBD rows for committee, aggregator and sync-contribution duties.

**Appendix A — Data sources** (non-normative)

- [ssv#3018](https://github.com/ssvlabs/ssv/pull/3018): proposer decision timing, 30 days mainnet (epochs 465250–472000), 25,126 duties, 99.6% Exporter trace coverage; round-1 duration distribution; per-cluster start drift.
- [ssv#2883](https://github.com/ssvlabs/ssv/pull/2883): companion analysis, 35 days, 44,838 duties; miss rate by decision-time band relative to the 4s deadline (0.05% below 3.0s, 1.9% at 3.0–3.5s, 30% at 3.5–4.0s, 100% past 4.0s).
- Projection method: decision times of round-1 decides are independent of the timeout (maximum round-1 duration 1,148ms); round-2 decisions move earlier by exactly the timeout reduction; miss probability by decision time relative to the deadline is taken from the two analyses above; newly induced round changes are simulated from per-operator start drift. The model reproduces today's 14 misses (13.9).
