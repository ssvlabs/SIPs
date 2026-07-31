| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Shane Moore | Early RANDAO Pre-Consensus Emission | Core | draft | [ePBS (EIP-7732) Support (#94)](https://github.com/ssvlabs/SIPs/pull/94) | 2026-07-23 |

[Discussion](https://github.com/ssvlabs/SIPs/discussions/100)

Depends on the ePBS SIP (currently [PR #94](https://github.com/ssvlabs/SIPs/pull/94)); per [SIP-0](sip0.md), this SIP cannot move to last-call before that SIP is approved, and is rejected if it is rejected (unless changed to remove the dependency).

**Summary**

Operators may emit their existing block-proposal RANDAO partial signature up to 2 slots before the proposal slot, so the cluster reconstructs `randao_reveal` before the slot starts instead of inside the post-Gloas ~3s block-production budget. No new duty, role, message kind, domain, topic, or container: the existing Proposer-duty `RandaoPartialSig` message is emitted earlier, stamped with the proposal slot as today. Within the maximum window, an early-emitting producer's recommended first attempt is at `slot_start(S - 1)`, with the ordinary in-slot path still invoked at `S`. Protocol changes are confined to message-validation timing, ordering, and duty-handling rules; receivers may additionally retain candidates locally when duty assignment is the only non-passing check. The rules activate by target proposal slot `S`: they apply when `epoch(S) >= GLOAS_FORK_EPOCH`, even if the permitted emission window for `S` begins before the fork in wall-clock time. The dependency on the ePBS SIP is activation coupling only.

**Motivation**

Gloas moves the attestation deadline to 1/4 slot. RANDAO pre-consensus (sign, gossip, collect 2f+1, reconstruct) is the first serial step of block production and costs a gossip round trip at slot start. The signed object, `SSZUint64(epoch)` under `DOMAIN_RANDAO`, is a pure function of the proposal slot's epoch, so it can be signed and exchanged at any time and a re-org can never invalidate it. Failure modes fall back to today's in-slot exchange, except at a receiver that already IGNOREd the early copy and did not retain it: gossip duplicate caching may suppress the byte-identical in-slot copy for the remainder of the useful duty window (see *Duty assignment* and *Optional duty-assignment retention*). Addresses [ssv-spec#373](https://github.com/ssvlabs/ssv-spec/issues/373).

**Rationale & Design Goals**

- No new duty: RANDAO always has an in-slot consumer (the Proposer duty), and a separate duty would still need the in-slot pre-consensus as its fallback. This differs from `ProposerPreferences` (defined in the ePBS SIP, [#94](https://github.com/ssvlabs/SIPs/pull/94)), which must reconstruct before the slot and is mutable.
- The 2-slot value is the maximum earliness receivers MUST support, not a gossip-latency requirement. Different producer schedules remain interoperable, but both reconstruction and third-party knowability begin only after `2f+1` shares, so the common producer policies determine typical readiness and exposure. A longer receiver window would widen reveal and stale-duty exposure without an identified need and would require protocol coordination.
- The recommended `S - 1` policy gives a full slot for gossip and reconstruction while reducing reveal exposure relative to emission near the receiver boundary. The SIP does not coordinate a second pre-slot attempt. Earlier and later policies remain conformant and are compared by the mandatory testnet measurement before mainnet activation guidance.
- A locally reported publication success does not prove peer delivery, and byte-identical re-publications may be gossip-deduplicated. The in-slot path is therefore a best-effort fallback, while reuse of cached reconstructions or signing artifacts is an implementation optimization.
- The per-(signer, slot) duplicate limit stays 1. Byte-identical copies share a content-derived gossip message identifier and are normally removed before application validation. Distinct operator-authenticated bytes for the same (`MessageID`, signer, slot) key normally have different gossip identifiers and are bounded by the existing one-message limit for each pre-consensus or post-consensus class.
- Because gossip duplicate-cache lifetimes can exceed the entire useful Early RANDAO window, a receiver that IGNOREs an early partial cannot rely on a byte-identical in-slot re-emission to recover that share. The dedicated clock tolerance narrows timing-induced loss, and optional duty-assignment retention can recover some missing or replaced-view cases. Remaining losses are bounded by the existing round-change path.

**Specification**

Notation: `S` is a proposal slot; `slot_start(s)` and `epoch(s)` as usual; for a non-genesis Gloas activation, `F` is the first slot of `GLOAS_FORK_EPOCH`; `SLOT_DURATION` is the network's beacon-chain seconds-per-slot (unchanged at Gloas, which alters only intra-slot timing). REJECT penalizes the delivering peer; IGNORE drops from ordinary processing without forwarding or penalty.

| Constant | Value | Description |
| -------- | ----- | ----------- |
| `EARLY_RANDAO_LEAD` | 2 slots | Maximum permitted emission lead and base receiver earliness allowance |
| `EARLY_RANDAO_RECOMMENDED_LEAD` | 1 slot | Recommended lead for an early producer's first publication attempt |
| `EARLY_RANDAO_CLOCK_TOLERANCE` | 1000 ms | Additional receiver allowance for honest clock disparity at the earliest emission boundary |

**Qualifying message**

A qualifying randao partial MUST satisfy all of:

- Proposer-role `MessageID`; type `RandaoPartialSig`; exactly one `PartialSignatureMessage` entry;
- `slot` = the proposal slot `S`; signed object `SSZUint64(epoch(S))` under `DOMAIN_RANDAO`, domain epoch `epoch(S)`;
- canonical SSZ; deterministic BLS share signature; deterministic RSA (PKCS#1 v1.5) operator signature; exactly one outer signer, equal to the embedded operator ID;
- eligibility predicate: `S >= EARLY_RANDAO_LEAD` and `epoch(S) >= GLOAS_FORK_EPOCH` (the Ethereum Gloas fork epoch, as used by the ePBS SIP, [#94](https://github.com/ssvlabs/SIPs/pull/94)).

The predicate is a pure function of `S`, evaluated identically by producers and receivers. It is not gated by the receiver's current epoch or by whether wall-clock time has reached the Gloas fork. The `S >= EARLY_RANDAO_LEAD` conjunct only keeps the producer's emission window well defined and excludes the first `EARLY_RANDAO_LEAD` slots at genesis.

For a non-genesis activation, `F` and `F + 1` are eligible target slots. Their producer windows begin, by the producer's clock, at `slot_start(F - 2)` and `slot_start(F - 1)`, respectively. There is no activation warm-up: the first Gloas target slot receives the full early window. Because receiver timing includes `EARLY_RANDAO_CLOCK_TOLERANCE`, implementations MUST enable the receiver rules no later than `slot_start(F - 2) - EARLY_RANDAO_CLOCK_TOLERANCE` by the receiver's clock. This is the earliest receiver-local time at which a conforming partial for `F` can pass timing validation.

The SSV fork schedule plays no part, and an emission window that spans an SSV fork activation is fine: every fork-sensitive artifact of a partial signature (gossip topic, domain, role gating) is derived from the message's stamped slot rather than from the moment it was emitted, so such a message is published, validated, and consumed entirely under the fork active at `S`. The same holds for a window spanning the Ethereum fork boundary. Containers violating the canonical form fall to existing structural rules (REJECT); BLS-share validity is not evaluated during message validation (see Message validation). Messages failing the predicate, and all non-randao messages, keep today's validation unchanged.

**Producer behavior**

For a locally known proposer duty at eligible slot `S`, an operator:

- MAY broadcast its qualifying randao partial at any wall-clock time in `[slot_start(S - EARLY_RANDAO_LEAD), slot_start(S))`;
- MUST NOT broadcast it before `slot_start(S - EARLY_RANDAO_LEAD)` by its own clock;
- if it elects to emit early and knows the duty by the recommended time, SHOULD make its first local publication attempt at `slot_start(S - EARLY_RANDAO_RECOMMENDED_LEAD)`, equivalent to `slot_start(S - 1)`;
- SHOULD still invoke the existing in-slot emission path at `S` regardless of its locally recorded early outcome; this is a best-effort fallback, not proof of redelivery, and a running origin's identical re-publish is absorbed by its own gossip layer (expected, not an error), while a restarted origin's re-publish aids recovery;
- MAY serve an already reconstructed reveal to the in-slot consumer without waiting for repeated local-share signing, envelope signing, or publication; implementations MAY reuse cached valid signing artifacts or schedule the emission attempt independently;
- MAY emit immediately for a duty first discovered after the recommended first-attempt time; SHOULD emit multiple eligible duties in ascending slot order (correctness does not depend on it).

For `S = F`, the interval begins at `slot_start(F - EARLY_RANDAO_LEAD)`, and the recommended producer time is `slot_start(F - 1)`, both before wall-clock Gloas activation. Producer scheduling therefore needs to run before `F` to follow the recommendation, but early emission itself remains optional.

Operators that never emit early remain fully conformant.

**Message validation**

Validation of a potential Early RANDAO message begins with the structural and canonical-form checks, which MUST run first; a structurally and canonically valid randao container satisfying the eligibility predicate is an Early RANDAO candidate, and the rules below apply to candidates only (all other messages keep today's validation unchanged). For a candidate, implementations MAY order operator-signature verification and the non-mutating contextual checks below (timing, slot ordering, duty assignment, duplicate limits) according to local denial-of-service policy, and MAY short-circuit on a contextual verdict that neither retains nor accepts the message. Any outcome that retains, accepts, forwards, feeds signature collection, or mutates ordinary validation state MUST first pass operator-signature verification. If a candidate fails both operator-signature verification and a non-retaining contextual check, either the contextual verdict or the signature-verification REJECT is conformant, but the candidate is never retained. Ordinary validation state (duplicate counts, signer state, slot high-water marks, and epoch counters) mutates only on acceptance or successful promotion, never on IGNORE or initial retention.

Candidate classification depends on message structure and the stamped target slot, not on whether a copy arrived before `slot_start(S)`. Consequently, the duty tri-state replaces existing RANDAO duty handling for every eligible target slot, including ordinary in-slot copies and later copies still within the existing Proposer lateness window.

Checks are staged: operator-signature verification gates retention, acceptance, forwarding, and state mutation; BLS-share validity gates consumption. A candidate is fully qualifying only once every applicable check has passed; honest producers emit qualifying messages by construction.

*Earliness.* A receiver MUST accept a candidate's timing iff

```text
slot_start(msg.slot) - local_now <= EARLY_RANDAO_LEAD * SLOT_DURATION + EARLY_RANDAO_CLOCK_TOLERANCE
```

inclusive on the accept side; strictly greater is IGNORE. This replaces the generic clock tolerance for this rule only. Lateness is unchanged (existing Proposer-role TTL against `msg.slot`). All other message classes keep their existing allowances.

`EARLY_RANDAO_CLOCK_TOLERANCE` is receiver-side headroom covering the assumed maximum 1 s pairwise honest clock disparity at the producer's legal boundary. It does not shift the recommended producer time.

*Slot ordering.* Candidate randao partials are exempt from the per-(operator, `MessageID`) highest-seen-slot rule in both directions: they MUST NOT be rejected for a slot below the high-water mark, and they MUST NOT advance the high-water mark applied to other message classes on the same `MessageID` (randao shares its `MessageID` with the proposal's consensus and post-consensus traffic). Final acceptance still requires the remaining qualifying-message checks; earliness, lateness, the duplicate limit of 1, and canonical form bound these messages instead.

*Duty assignment.* Tri-state, evaluated per candidate against the receiver's proposer-duty view for `epoch(msg.slot)` and the message's validator:

- Known (a retained view is authoritative for the candidate; see below) and assigned at exactly `msg.slot`: the duty-assignment check passes.
- Known and not assigned: the duty-assignment verdict is IGNORE, never REJECT (receiver views can be stale or re-orged); an independently failing rule may still determine the final verdict under the ordering allowance above.
- Unknown (otherwise): IGNORE.

Either duty-assignment IGNORE is eligible for the optional local retention below.

A proposer-duty view is authoritative for an epoch iff the receiver retains a complete schedule containing exactly `SLOTS_PER_EPOCH` duties, one for every distinct slot in that epoch. Such a view is authoritative for every validator in the epoch, independent of the receiver's local SSV registry. A schedule that fails this shape establishes no new view. A newly retained complete schedule replaces the prior view; a failed or malformed refresh and a local validator-set change do not invalidate a previously retained authoritative view. A reorg likewise does not revoke authority until a replacement complete schedule is retained, so a stale view remains Known with the loss class described below.

Receivers SHOULD hold next-epoch proposer duties by the tail of each epoch.

A reorg that changes the proposer-duty dependent root can leave a retained complete schedule stale. It can therefore yield Known-and-not-assigned and IGNORE an honest early share. Optional retention can preserve the candidate for full revalidation after replacement; otherwise, duplicate suppression may prevent recovery and the existing round-change path remains the fallback. Receivers SHOULD refresh proposer duties promptly after detecting a change.

**Optional duty-assignment retention**

A receiver MAY retain an Early RANDAO candidate whose only non-passing check is duty assignment, whether its duty state is Unknown or Known-and-not-assigned. Retention is local implementation policy and does not change the authoritative view or the gossip verdict, which remains IGNORE. A receiver that does not retain the candidate remains conformant.

Before retention, the candidate MUST pass operator-signature verification and every applicable message-validation rule other than duty assignment. Initial retention MUST NOT forward the original message, penalize the delivering peer, feed the candidate to signature collection or any downstream reconstruction cache, or mutate ordinary validation state.

If a retained candidate is later considered for promotion, it MUST be reprocessed as if newly received under the receiver's then-current duty view, with every stateful rule applied at that time. It may be accepted and consumed only if full revalidation succeeds. If duty assignment remains its only non-passing check, it MAY remain retained under local policy; if any other applicable rule fails, it MUST be discarded. Promotion MUST NOT retroactively forward the original message or change its original gossip verdict. An implementation may attempt promotion before or after `S`; the ordinary lateness rule determines whether the candidate can still be accepted. Capacity, keying, duplicate handling within the retention store, eviction, expiry, persistence, and earlier discard are implementation-specific.

Shares that are neither promoted nor normally accepted MUST NOT be fed to signature collection or any downstream reconstruction cache.

**Reconstruction and consumption**

Reconstruction semantics are unchanged. A completed reconstruction MAY be served immediately to the in-slot consumer; invoking the best-effort in-slot emission path does not require the consumer to wait for repeated signing or publication. The reconstructed reveal is a per-epoch value: an implementation MAY serve a proposal at slot `Y` from a reconstruction built via shares stamped for slot `X` of the same epoch, provided every contributing share was promoted or normally accepted. Wire messages remain per-slot-stamped.

"Promoted or normally accepted" is evaluated at receipt or promotion time against the receiver's then-current duty view and is never re-evaluated at consumption. The cross-slot case is reachable two ways: a validator with two proposals in the same epoch, and a reorg that moves a proposal from `X` to `Y` after `X`-stamped shares were accepted under the pre-reorg view; the allowance exists so a completed early collection survives the shift. The signed object is identical for `X` and `Y` (`SSZUint64(epoch)`), so reuse has no cryptographic effect; implementations that key collection by signing root exercise it naturally, and implementations that require exact-slot consumption remain conformant.

**Test expectations**

Cross-client vectors MUST cover:

- earliness boundary at exactly `EARLY_RANDAO_LEAD * SLOT_DURATION + EARLY_RANDAO_CLOCK_TOLERANCE` (accept) and beyond (IGNORE);
- candidate scope: an eligible in-slot `RandaoPartialSig` under an Unknown duty view produces the same initial IGNORE verdict as an otherwise-identical early copy;
- message-kind scoping: a non-RANDAO Proposer-role message two slots early remains outside the candidate path and receives the existing timing IGNORE without the Early RANDAO ordering exemption;
- Known-assigned passes, while Known-unassigned and Unknown each produce the same initial IGNORE gossip verdict whether optional retention is enabled or not;
- the stale-Known reorg path: an honest share is IGNOREd as Known-and-not-assigned under a stale complete schedule, and a later replacement schedule does not retroactively change the original gossip verdict;
- same-epoch duty move: shares stamped `X` accepted under a duty-at-`X` view, the duty moves to `Y` in the same epoch, and the proposal at `Y` completes (served from the `X` reconstruction where the implementation reuses; via the in-slot exchange otherwise);
- the two-direction ordering exemption with proposals at consecutive slots for one validator;
- Known authority: a retained schedule with exactly `SLOTS_PER_EPOCH` duties over distinct in-epoch slots is authoritative for every validator; a missing, duplicate-slot, or out-of-epoch entry establishes no new view; local validator-set changes and failed or malformed refreshes do not invalidate a prior authoritative view;
- an operator-authenticated share whose BLS signature is invalid is discarded at consumption and contributes to no reconstructed reveal; reconstruction still succeeds from a valid threshold of honest shares;
- the eligibility predicate at genesis: slots below `EARLY_RANDAO_LEAD` are ineligible even when `GLOAS_FORK_EPOCH` is 0;
- activation gating: `epoch(S)` immediately before `GLOAS_FORK_EPOCH` is ineligible; for a non-genesis `F`, an otherwise qualifying `msg.slot = F` received at receiver-local time `slot_start(F - 2) - EARLY_RANDAO_CLOCK_TOLERANCE` and `msg.slot = F + 1` received at `slot_start(F - 1) - EARLY_RANDAO_CLOCK_TOLERANCE` enter the Early RANDAO validation path even though local wall-clock time is before the fork; an SSV fork scheduled anywhere does not change that;
- multi-fault precedence: a structurally valid message failing both operator-signature verification and a contextual check that local policy does not retain (e.g. invalid signature plus a non-retained duty-assignment IGNORE, or invalid signature plus too-early) may produce either the contextual verdict or REJECT, but is never retained, accepted, forwarded, collected, or state-mutating;
- late mesh join and cross-epoch stamps.

Implementations that provide optional retention SHOULD additionally test that initial retention leaves the gossip verdict and ordinary validation state unchanged, candidates with invalid operator signatures are never retained, unpromoted candidates never reach signature collection, promotion performs full revalidation, a repeated duty-assignment-only IGNORE follows local retention policy rather than mandatory failure discard, and any other revalidation failure discards the candidate. Implementations that retain Known-and-not-assigned SHOULD also test promotion after a replacement schedule assigns the candidate without changing its original gossip verdict.

Producer implementations that emit early SHOULD test the recommended `S - 1` attempt, an injected local publication failure followed by the in-slot path, duty discovery after the recommended time, unconditional invocation of the in-slot path with and without a completed reconstruction, and completion of a cached in-slot consumer without waiting for repeated signing or publication.

Before mainnet activation guidance, testnet measurement MUST quantify early-share miss incidence at duty start (stratified by emission lead, receiver restarts, partition duration, leader round, and whether receiver retention was enabled), record promotion timing and outcome where retention is enabled, and MUST demonstrate round-2 viability under Gloas timing, meaning block publication before the post-Gloas useful-block deadline. The emission-lead comparison MUST include `EARLY_RANDAO_RECOMMENDED_LEAD`, at least one earlier lead, and at least one later lead, and MUST record threshold-ready time relative to `slot_start(S)` so the resulting reveal-exposure window is explicit.

**Security Considerations**

- Deliberately ineligible early emission only starves the attacker's own share (only the signer can produce the bytes). The clock tolerance and IGNORE-not-REJECT choices narrow the permanent-loss class for honest shares, and optional retention can further narrow duty-view losses. Candidates that are not retained or are discarded before promotion remain residual loss cases.
- Gossip duplicate-cache lifetimes can exceed the entire useful Early RANDAO window, so a receiver that has already seen and IGNOREd an early share cannot rely on byte-identical re-emission for recovery. A receiver restart can both clear its seen state and drop any in-memory retained candidates; restarts and partitions are correlated events. Residual: a live round-1 leader without a reconstruction forces a round change. Mitigations: later emission inside the window, unconditional in-slot emission where duplicate state does not suppress it, optional retention, and the mandatory testnet measurement above.
- Under `EARLY_RANDAO_RECOMMENDED_LEAD`, a homogeneous cluster may make the reveal knowable close to one slot before `S`, compared with roughly 2-4 s today. If at least `2f+1` operators emit at the earliest legal boundary, the reveal may instead become knowable close to two slots before `S`; `EARLY_RANDAO_CLOCK_TOLERANCE` is receiver-local acceptance headroom, not additional producer lead. The wider reaction window enables adaptive bribery, censorship, or targeted DoS when withholding the proposal favors an adversary (one randao bit per affected slot, compounding across slots). It is bounded by the existing single-proposal grinding bound and can be adjusted via producer policy without changing the receiver window.
- Optional retention exposes local signature-verification, memory, and revalidation work. Retaining Known-and-not-assigned increases candidate supply because it does not require a missing view. Implementations are responsible for bounding local resource use and MAY scope retention to committees they participate in. Capacity, eviction, and persistence choices do not affect wire conformance.
- No unauthenticated retention amplification: messages failing operator-signature verification are never forwarded or retained. An operator-authenticated share whose BLS signature proves invalid at consumption is attributable and MUST be discarded without contributing to a reconstructed reveal. A signer emitting two distinct containers for one (signer, slot) is provably misbehaving and attributable via its operator signature.
- No slashing surface: the randao object is not slashable and is a pure function of public data; early signing changes when it is signed, not what.
