| Author | Title | Category | Status | Date |
| ------ | ----- | -------- | ------ | ---- |
| Gal Rogozinski ([@GalRogozinski](https://github.com/GalRogozinski)) | QBFT Round Timeouts for Glamsterdam | Core | draft | 2026-09-06 |

**Summary**

Glamsterdam (ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)) moves the beacon attestation deadline from 4s to 3s into the slot. The proposer duty's QBFT round timeout is 2s from instance start ([SIP-6](./constant_qbft_timeout.md)), and a proposer instance starts only after RANDAO pre-consensus and block retrieval, typically 1.1–1.5s into the slot. Under the new deadline a round change therefore starts round 2 after attesters have already voted: a round change (~0.18% of proposer duties) goes from usually recoverable to usually fatal, roughly tripling missed proposals (0.056% → 0.166%).

This SIP lowers the proposer duty's quick round timeout from 2000ms to 1500ms and keeps the timer anchored at instance start. It changes nothing else: no message-validation rule changes, and the other duties' timers are out of scope.

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

The 1250ms and 1000ms rows are within the 95% interval of the 1500ms row (22–32/month). Round 1, when it succeeds, is fast: median 59ms, p99 191ms, slowest success in 30 days 1,148ms. A 1500ms timer never fires on a round 1 whose own duration would have fit (352ms of headroom above the observed maximum); a 1000ms timer would fire on 5–12 real duties a month. Start drift across operators still induces an estimated 1–2 round changes a month at 1500ms — see Security Considerations.

*Design goals, in priority order:*

1. A round change MUST leave round 2 enough time to decide and broadcast before the 3s attestation deadline for clusters that start consensus by ~1.3s.
2. The timer MUST NOT fire on a round 1 that would have succeeded: the timeout stays above the observed maximum round-1 duration with margin.
3. The round-1 budget MUST stay independent of `ProposerDelay`, so the timer remains anchored at instance start, not slot start. A slot-anchored deadline short enough to help typical clusters would fire before round 1 even begins for clusters using a deliberate delay.
4. No change to message formats, signing, or domains; pre-upgrade and post-upgrade operators MUST interoperate within one cluster.

**Specification**

*Definitions*

- `slot_start(height)`: **wall-clock** time, beacon genesis plus `height × SECONDS_PER_SLOT`. `height` is the message's `Height` field, which is the slot (ssv-spec [`ssv/runner.go`](https://github.com/ssvlabs/ssv-spec/blob/main/ssv/runner.go#L247), [`ssv/committee.go`](https://github.com/ssvlabs/ssv-spec/blob/main/ssv/committee.go#L200)).
- `round_entered_at(round)`: a **monotonic** local clock reading taken when the instance entered `round`. For `round = 1` this is the instance start; for `round > 1` it is the moment the instance advanced to that round (on timeout, or on the `f+1` partial-quorum rule below). It MUST NOT be derived from wall-clock time, so that an NTP step cannot shorten or extend a running round. On process restart or mid-round re-sync the instance re-enters its current round and the timer restarts from the new `round_entered_at`.
- *`f+1` partial-quorum round advance*: an instance that receives `RoundChange` messages for rounds above its own from `f+1` distinct operators adopts the lowest such round and re-arms its timer, without waiting for its own timeout. This is ssv-spec behaviour ([`qbft/round_change.go`](https://github.com/ssvlabs/ssv-spec/blob/main/qbft/round_change.go#L87-L112)), not a SIP; it is stated here because the mixed-version argument below depends on it.
- `ProposerRoundTimeout(round)`: the duration after `round_entered_at(round)` at which a proposer instance MUST act on expiry (the existing `UponRoundTimeout` behaviour; unchanged). Defined below.

*Proposer duty*

```go
var (
    quickTimeoutThreshold = Round(8)                // unchanged (SIP-6)
    proposerQuickTimeout  = 1500 * time.Millisecond // new: was quickTimeout (2s)
    slowTimeout           = 2 * time.Minute         // unchanged (SIP-6)
)

// ProposerRoundTimeout returns the duration until the next timeout for a
// proposer round. The timer is armed on entering the round and is relative to
// that entry, not to slot start.
//
// Only the proposer role is specified here. The committee, aggregator and
// sync-committee-contribution timers are slot-anchored — round r expires at
// slot_start + head_start(role) + quickTimeout * r — and are unchanged by this
// SIP; see Open Questions.
func ProposerRoundTimeout(r Round) time.Duration {
    if r > quickTimeoutThreshold {
        return slowTimeout
    }
    return proposerQuickTimeout
}
```

This amends [SIP-6](./constant_qbft_timeout.md)'s `RoundTimeout`, which returned `quickTimeout` for every role. SIP-6's value is retained for the roles this SIP does not cover.

- An operator running a proposer instance MUST arm a timer of `ProposerRoundTimeout(round)` upon entering each round. On expiry, if it is still in `round` and `round < 2`, it MUST broadcast `RoundChange(round + 1)`.
- The proposer timer MUST be anchored at `round_entered_at`, never at `slot_start`.
- A proposer instance MUST NOT progress past round 2. No round after 2 can land a block under a 3s attestation deadline, and ssv message validation has capped proposer messages at round 2 since v2.4.3 ([ssv#2807](https://github.com/ssvlabs/ssv/pull/2807)), so higher rounds are unreachable for this role in any case. `slowTimeout` is therefore dead code for the proposer.
- On round-2 expiry the instance MUST NOT broadcast `RoundChange(3)` and MUST NOT arm a further timer. It remains in round 2 until it decides or the duty ends.

*Timer cases* (proposer role)

| # | Situation | Timer armed | On expiry |
| - | --------- | ----------- | --------- |
| 1 | Instance starts round 1 at `t0` | expires `t0 + 1500ms` | broadcast `RoundChange(2)`, enter round 2 |
| 2 | Round 2 entered on round-1 timeout at `t1` | expires `t1 + 1500ms` | no `RoundChange(3)`; stay in round 2 |
| 3 | Round 2 entered at `t2` on `f+1` `RoundChange(2)` from other operators, before this node's own round-1 timer fired | expires `t2 + 1500ms` | no `RoundChange(3)`; stay in round 2 |

Cases 2 and 3 arm the same duration from different instants: each round's timer is measured from that operator's own `round_entered_at`, so a node dragged into round 2 early by case 3 also leaves round 2 early. Neither case re-arms after expiry.


*Compatibility and rollout*

- No message, signature or domain change. Pre-upgrade and post-upgrade operators MAY coexist in one cluster.
- Round-change quorum in a mixed cluster: upgraded operators broadcast `RoundChange(2)` at `start + 1.5s`, non-upgraded at `start + 2s`. Once `f + 1` operators in a cluster are upgraded, the existing partial-quorum rule advances the rest at 1.5s; before that the cluster behaves as today. The change is never worse than the status quo during rollout.
- This is **not a fork**. Per [SIP-7](./fork_support.md) a fork is a non-backwards-compatible change, and there is none here: no message, signature or domain change, and a mixed cluster stays live throughout. No fork assignment is required.
- Rollout condition instead: the change SHOULD be active on a majority of mainnet operators before the Glamsterdam activation epoch. A cluster that is only partly upgraded behaves as it does today.

**Security Considerations**

- *Consensus safety.* QBFT safety does not depend on timeout values; round-change justification and message validation semantics are unchanged. No impact.
- *Liveness.* A shorter round-1 window tolerates a slower honest leader less. The bound is empirical: 352ms above the slowest round-1 success observed in 30 days across 24,984 round-1 decisions, with simulated newly induced round changes of ~1–2 per month network-wide. Round 2 remains available. Under mixed versions liveness is bounded below by today's behaviour.
- *Slashing.* Unchanged. An instance decides at most one value and the block is signed only after the decision; a round change never yields a second signed block for the same slot.
- *Message validation surface.* None. This SIP no longer changes any message-validation rule: ssv exempts the proposer from the round-spread check, and the round cap is unchanged.
- *Residual risk.* The whole change is a 500ms-narrower round-1 window. The margin is empirical: the slowest round-1 success observed in 30 days was 1,148ms, so a 1500ms timer leaves 352ms of headroom and never fires on a round 1 whose own duration would have fit. That is a statement about round-1 duration, not about the duty as a whole — per-operator start drift still induces an estimated 1–2 round changes a month that a 2000ms timer would not have, which the liveness bullet above quantifies. A 1000ms timer would additionally have fired on 5–12 round 1s a month that did succeed, which is why the value is not lower.
- *Operator incentives.* The instance-start anchor keeps `ProposerDelay` an operator choice. Under a 3s deadline a delayed cluster that round-changes misses regardless of this SIP (`~2.0s + 1.5s > 3s`); operators SHOULD keep `ProposerDelay` at or below the existing 1s cap.

**Alternatives Considered** (non-normative)

1. *Slot-anchored proposer timer* ([ssv#2429](https://github.com/ssvlabs/ssv/issues/2429), the approach of [SIPs#37](https://github.com/ssvlabs/SIPs/pull/37) for other duties). To help typical clusters under a 3s deadline the round-1 deadline would sit at ~1.5–2.0s from slot start, which is before or at round-1 start for clusters using `ProposerDelay ≥ 0.5s`, forcing a round change on every such duty. Deferred; worth revisiting if ePBS bid retrieval shortens consensus start enough to remove the case for `ProposerDelay`.
2. *1250ms or 1000ms.* Projected gain over 1500ms is inside the error bars; 1000ms would time out 5–12 real round-1s per month.
3. *No change.* Missed proposals roughly triple (0.056% → 0.166%).

**Open Questions**

- ePBS replaces relay payload fetch with builder-bid selection; if consensus start drops well below 1s the slot-anchored alternative becomes attractive. Re-measure after the fork.
- The committee, aggregator and sync-committee-contribution timers are slot-anchored with a head start pinned to the old beacon deadlines, so Glamsterdam leaves them stale. Out of scope here; needs its own SIP.

**Appendix A — Data sources** (non-normative)

- [ssv#3018](https://github.com/ssvlabs/ssv/pull/3018): proposer decision timing, 30 days mainnet (epochs 465250–472000), 25,126 duties, 99.6% Exporter trace coverage; round-1 duration distribution; per-cluster start drift.
- [ssv#2883](https://github.com/ssvlabs/ssv/pull/2883): companion analysis, 35 days, 44,838 duties; miss rate by decision-time band relative to the 4s deadline (0.05% below 3.0s, 1.9% at 3.0–3.5s, 30% at 3.5–4.0s, 100% past 4.0s).
- Projection method: decision times of round-1 decides are independent of the timeout (maximum round-1 duration 1,148ms); round-2 decisions move earlier by exactly the timeout reduction; miss probability by decision time relative to the deadline is taken from the two analyses above; newly induced round changes are simulated from per-operator start drift. The model reproduces today's 14 misses (13.9).
