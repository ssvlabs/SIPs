|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Shane Moore    | ePBS (EIP-7732) Support    | Core       | draft               | 2026-04-18 |

## Summary

Describes the SSV spec changes needed to keep SSV operators performing validator duties correctly after ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732), is implemented in Ethereum's consensus layer Gloas fork. Based on the pinned [Gloas consensus-spec snapshot](https://github.com/ethereum/consensus-specs/tree/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas) (`ethereum/consensus-specs@a5a1bc630`, reviewed 2026-08-26), which includes the progressive Gloas types from [EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) via [consensus-specs #4630](https://github.com/ethereum/consensus-specs/pull/4630).

Validator client related changes via ePBS:
1. earlier slot deadlines
2. attestation handling changes, including preservation of the Gloas `attestation.data.index` semantics
3. the new Payload Timeliness Committee (PTC)
4. the Gloas proposer flow using `produceBlockV4`. The SSV cluster decides the `Gloas.BeaconBlock` together with the self-build `payload_root` and, in one post-consensus round, signs the block and, on the self-build path, the `SignedExecutionPayloadEnvelope` ([§4](#4-modified-proposer-duty), [§6](#6-envelope-signing-self-build-path)).
5. the new `SignedProposerPreferences` message must be submitted if the node operator wants to be able to select block bids received over p2p
6. clusters opting into direct builder connections sign `BuilderRequestAuth` per builder as an extension of the proposer-preferences duty ([§5](#5-proposer-preferences-duty)), authenticating the cluster's builder requests in [§4](#4-modified-proposer-duty)
7. the existing validator-registration duty is deprecated at the Gloas fork, its purpose replaced by `SignedProposerPreferences` ([§5](#5-proposer-preferences-duty))
8. EIP-7688 changes Gloas aggregate, block, and envelope signing roots without changing their SSZ serialization

## Motivation

Gloas changes validator duties in ways that break a few current SSV assumptions:

- attestation `index` is no longer safely reconstructible from local validator duty data
- the new PTC duty has a late in-slot deadline
- SSV validators must broadcast `SignedProposerPreferences` or they cannot accept builder bids for their slots
- matching SSZ bytes can conceal a progressive-versus-positional signing-root split between operators

## Rationale

Key design choices and why:

- **`BeaconVote` gains `AttestationDataIndex` at the Gloas fork.** In Gloas, `AttestationData.Index` is BN-supplied and part of the signed attestation root, so it must travel through QBFT consensus data rather than being reconstructed locally. The pre-Gloas encoding stays frozen for pre-Gloas slots, so the two are mutually rejecting on length.
- **PTC is a validator-scoped, non-QBFT runner.** Each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% broadcast mark, validates incoming partial signatures against its own derived signing root, and reconstructs when that root reaches threshold, the same one-round shape as `ProposerPreferences` ([§5](#5-proposer-preferences-duty)). A PTC vote is one beacon node's observation, not a value to negotiate, so QBFT would only add round-trips that risk the late-slot deadline ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)).
- **Proposer-preferences is validator-scoped and non-QBFT.** The per-validator `fee_recipient` is configured cluster-side and is cluster-consistent in practice; `target_gas_limit` is operator-configured, with a client default when unset; operators in a cluster must agree byte-for-byte on the value used at signing time, the same convergence requirement as the existing validator-registration flow. Operators independently derive the full `ProposerPreferences` and validate incoming partial signatures against their own derived signing root; reconstruction succeeds only when a quorum of operators converge on one signing root. The registration-like one-round partial-sig-and-submit flow from `voluntary_exit.md` fits directly.
- **Builder request auth rides the proposer-preferences duty.** The auth signing round has the same cadence as [§5](#5-proposer-preferences-duty): lookahead-driven, per proposal slot, re-triggered by the same duty-refresh events. It therefore reuses `RoleProposerPreferences` with a second partial-signature type rather than a new role. Each distinct auth signing root travels in its own single-entry packet, preserving [§7](#7-ssv-message-validation)'s one-entry packet rule and isolating per-builder failures: a divergent entry costs that builder's bids, never the slot.
- **Block QBFT decides the `Gloas.BeaconBlock` plus the self-build `payload_root`, and envelope signing rides its post-consensus round.** Once [§4](#4-modified-proposer-duty) decides, `bid.block_hash` and `bid.execution_requests_root` pin exactly one valid envelope, so there is no value to negotiate; `payload_root` is the only envelope field the block does not commit to, and carrying those 32 bytes in the decided value leaves nothing to distribute. Each operator signs the blinded envelope root ([§6](#6-envelope-signing-self-build-path)) as a second entry of its post-consensus packet and the builder operator publishes.
- **EIP-7688 changes Ethereum object roots without changing their SSZ bytes.** Gloas `Attestation`, `BeaconBlockBody`, `ExecutionPayload`, `ExecutionRequests`, `ExecutionPayloadEnvelope`, and their progressive list fields use progressive merkleization; SSV's outer QBFT and pubsub containers, including the decided `DataSSZ` bytes, do not change. The affected SSV signing roots are exactly the [§2](#2-modified-attestation-duty) aggregate, the [§4](#4-modified-proposer-duty) block, and the [§6](#6-envelope-signing-self-build-path) envelope. Every other object SSV signs (`AttestationData`, selection proofs, the RANDAO epoch, sync-committee message roots and `ContributionAndProof`, `VoluntaryExit`, `PayloadAttestationData`, `ProposerPreferences`, `BuilderRequestAuth`) keeps its existing merkleization, and duties that consume a beacon-node-supplied `beacon_block_root` need no new SSV rule.

## Specification

### 1. Slot Timing Changes

All existing validator duty deadlines shift earlier in the slot. A new PTC deadline is added.

Relevant consensus-spec references:

- [Validator time parameters](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#time-parameters)

| Duty | Pre-ePBS | Post-ePBS (Gloas) |
|------|----------|--------------------|
| Attestation | 1/3 slot (~4s) | 1/4 slot (25%, ~3s) |
| Sync Committee Message | 1/3 slot | 1/4 slot (25%) |
| Aggregation | 2/3 slot (~8s) | 1/2 slot (50%, ~6s) |
| Sync Committee Contribution | 2/3 slot | 1/2 slot (50%) |
| PTC Attestation | - | 3/4 slot (75%, ~9s) |
| Envelope reveal (self-build, [§6](#6-envelope-signing-self-build-path)) | - | 1/2 slot (50%, ~6s) |

### 2. Modified Attestation Duty

Relevant consensus-spec references:

- [Validator attestation changes](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#attestation)
- [Gloas `Attestation` progressive container](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/beacon-chain.md#attestation)

#### Consensus-spec change

Under Gloas, the attestation `index` field no longer carries the beacon committee index:

- if attesting to a same-slot block, set `index = 0`
- otherwise, for a non-same-slot attestation:
  - `index = 0` means payload `EMPTY`
  - `index = 1` means payload `FULL`

This value is fork-choice dependent and is supplied by the beacon node in the `AttestationData` returned to the validator client.

#### Why the current SSV model is insufficient

SSV currently omits `AttestationData.Index` from `BeaconVote` and fills it locally at reconstruction (`0` on the current Electra/Fulu path, duty-derived earlier). Post-Gloas this is unsafe: `Index` is BN-supplied and part of the signed attestation root, so if SSV drops it from consensus data, operators can agree on the same `block_root/source/target` and still sign the wrong root.

#### Required change

At the Gloas fork, `BeaconVote` is extended with an `AttestationDataIndex` field (`phase0.CommitteeIndex`, a `uint64` alias matching `AttestationData.Index`, so reconstruction is a direct field assignment). The pre-Gloas 3-field encoding stays frozen for pre-Gloas slots; decoders select by fork, and the two encodings are mutually rejecting on length (fixed-size SSZ, 112 vs 120 bytes). Implementations MAY realize the transition as two concrete types. The restricted Gloas value space (`0` = `EMPTY`, `1` = `FULL` for non-same-slot attestations; `0` for same-slot) is enforced in the value check below, not at the type level.

```go
// Pre-Gloas encoding (ssv-spec types/consensus_data.go); frozen for pre-Gloas slots
type BeaconVote struct {
    BlockRoot phase0.Root `ssz-size:"32"`
    Source    *phase0.Checkpoint
    Target    *phase0.Checkpoint
}

// Gloas encoding; implementations may keep a distinct type through the transition
type GloasBeaconVote struct {
    BlockRoot            phase0.Root        `ssz-size:"32"`
    Source               *phase0.Checkpoint
    Target               *phase0.Checkpoint
    AttestationDataIndex phase0.CommitteeIndex // copied from AttestationData.Index
}
```

#### Value check

A new `GloasBeaconVoteValueCheckF()` mirrors today's `BeaconVoteValueCheckF()` and additionally:

- rejects `AttestationDataIndex` values other than `0` or `1`;
- builds the `AttestationData` passed to `IsAttestationSlashable` using the decided `AttestationDataIndex` rather than the existing `math.MaxUint64` sentinel, so the Gloas double-vote predicate trips correctly when an operator is asked to sign both `index=0` and `index=1` for the same `(source, target, slot)`.

The existing sentinel is in place because pre-Gloas consensus data carries no `CommitteeIndex`; `math.MaxUint64` keeps `IsAttestationSlashable` from flagging legitimate same-`(source, target, slot, BlockRoot)` attestations as double-votes.

The [Gloas same-slot rule](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#attestation) (`block.slot == data.slot ⇒ data.index = 0`) has no beacon-node check on the value-check path: the cluster has only the QBFT-decided `BlockRoot` and trusts `AttestationDataIndex` from the leader. An operator MAY reject a value with `AttestationDataIndex = 1` when its own view already establishes, at validation time, that the block at `BlockRoot` has slot `duty.slot` (for example from a head event naming that root and slot); the rejection induces a round change instead of a network-rejected attestation. The check has no false positive (a reorg never changes a block's slot, and an honest beacon node never yields `index = 1` for a same-slot block), but it MUST be evaluated against knowledge fixed when the instance starts and never re-evaluated with later knowledge: QBFT re-runs the value check on justified re-proposals and decided values, and an operator that learned the block's slot mid-instance would otherwise reject a value it had already prepared. Operators without such a view skip the check. A bad same-slot `index=1` that escapes it is rejected by the ethereum network and ignored on chain but is not slashable, while cross-`index` equivocation over the same `(source, target, slot, BlockRoot)` is still caught by `IsAttestationSlashable` per the previous bullet.

Pre-Gloas slots continue to run `BeaconVoteValueCheckF()` unchanged.

#### Implementation note: aggregation path

The `BNRoleAggregator` duty (handled by the aggregator-committee runner) fetches aggregated attestations from the Beacon API's aggregate-attestation endpoint with `attestation_data_root` as an input. Implementations must compute that root over the full Gloas `AttestationData` reconstructed from the decided `GloasBeaconVote` (including `AttestationDataIndex`), rather than from a locally reconstructed pre-Gloas shape; a root computed without the decided `index` matches no aggregate.

`AggregatorCommitteeConsensusData.Version` is part of the QBFT-decided value, so operators must stamp it identically: use `DataVersionGloas` at Gloas slots (as [§4](#4-modified-proposer-duty) does for `ProposerConsensusData`).

The aggregate's SSZ serialization is unchanged by EIP-7688, but its merkleization is not. At Gloas slots, `Attestation` is a four-field `ProgressiveContainer` and `aggregation_bits` is a `ProgressiveBitList`. `AggregateAndProof` remains an ordinary container, but its root commits to the nested progressive `Attestation` root. Both the aggregator-committee path and any retained single-validator aggregation path MUST therefore decode the aggregate as the Gloas type and compute the `AggregateAndProof` signing root under `DOMAIN_AGGREGATE_AND_PROOF` using Gloas merkleization. Treating Gloas bytes as an Electra aggregate produces an obsolete positional root.

### 3. New Duty: Payload Timeliness Committee (PTC) Attestation

Relevant consensus-spec references:

- [Validator payload timeliness attestation flow](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#payload-timeliness-attestation)
- [Beacon-chain payload attestation containers](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/beacon-chain.md#payloadattestationdata)
- [Fork-choice payload attestation deadline](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/fork-choice.md#new-get_payload_attestation_due_ms)

PTC is a per-slot consensus-layer-selected set of validators that attests to payload and blob availability for the slot's beacon block.

Each validator signs a `PayloadAttestationData` object carrying `beacon_block_root`, `slot`, `payload_present`, and `blob_data_available`, then submits a validator-specific `PayloadAttestationMessage(validator_index, data, signature)` to the beacon node via [`POST /eth/v1/beacon/pool/payload_attestations`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/beacon/pool/payload_attestations.yaml).

At the start of each epoch, SSV should fetch PTC duties for the current and next epoch via [`POST /eth/v1/validator/duties/ptc/{epoch}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/duties/ptc.yaml) (the endpoint serves at most one epoch ahead; the current-epoch fetch covers operators starting mid-epoch) and refresh them on duty-dependent-root changes. On a dependent-root change, the new response is authoritative for that epoch: cached duties for the epoch are replaced rather than merged, since a merge could retain assignments that no longer exist under the new root.

Two distinct deadlines bound the runner. `PAYLOAD_DUE_BPS = 50%` is the payload-side observation cutoff: `payload_present` is the predicate "a `SignedExecutionPayloadEnvelope` for `beacon_block_root` was seen before the 50% mark" (a first-seen-time test, not a "present now" query), so an envelope arriving after the cutoff never flips it to `True` and the predicate is frozen for the rest of the slot. `blob_data_available` has no such upstream cutoff; it is `is_data_available(beacon_block_root)` at whatever time the data is evaluated. `PAYLOAD_ATTESTATION_DUE_BPS = 75%` is the consensus-spec-recommended broadcast time, a soft target leaving ~25% of the slot for propagation to the next slot's proposer. The hard deadline is slot end (100%): a slot-N payload attestation is accepted on the wire only while `data.slot == current_slot` and is consulted by fork choice only in slot N+1, so nothing consumes a slot-N vote during the [75%, 100%] window. Each operator therefore evaluates `PayloadAttestationData` (`payload_present`, `blob_data_available`, `beacon_block_root`) from its beacon node at the 75% broadcast mark, by which point `payload_present` has been frozen for a quarter slot, and runs its partial-signature round in the otherwise-free [75%, 100%] window. Broadcast may complete after 75% and still propagate; past slot end the message is dropped (gossip IGNORE and fork-choice wire REJECT when `data.slot != current_slot`), and each missed vote chips at the `PTC_SIZE/2` threshold that governs whether fork choice extends the payload.

The beacon node also emits SSE events a runner may consume as a push trigger for when to evaluate its observation, instead of polling toward the cutoff: `execution_payload_gossip` and `execution_payload` fire when a `SignedExecutionPayloadEnvelope` passes `execution_payload`-topic gossip validation and when it is imported into fork choice, and `execution_payload_available` fires once the node has verified the payload and blobs are available and ready for payload attestation ([beacon-APIs event stream](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/eventstream/index.yaml)). These are timing signals only: `payload_present` and `blob_data_available` still come from the BN-computed `PayloadAttestationData` fetched via [`GET /eth/v1/validator/payload_attestation_data?slot={slot}`](https://github.com/ethereum/beacon-APIs/blob/4b4d89a20254d05e2b94c75c8cfd5170ccb77a36/apis/validator/payload_attestation_data.yaml) (`slot` moved from path to query parameter after `v5.0.0-alpha.2`).

There is no pre-consensus phase and no QBFT round. Each operator evaluates `PayloadAttestationData` from its own beacon node at the 75% broadcast mark and signs that observation directly; there is no leader and no negotiated value. Because a per-validator BLS signature reconstructs only from partial signatures over byte-identical data, each operator pins its 75% `PayloadAttestationData` snapshot (with `slot` taken from the duty) and validates and aggregates incoming partial signatures against exactly that signing root.

An operator that has seen no beacon block for the slot abstains (submits nothing), matching the Gloas validator spec; the payload-attestation-data endpoint signals this case explicitly with a `204` response ("no block has been seen for the requested slot"). Otherwise, the operator's runner for each of its local PTC-assigned validators produces one partial signature over the full `PayloadAttestationData` under `DOMAIN_PTC_ATTESTER` (domain epoch = `compute_epoch_at_slot(duty.slot)`), because each `PayloadAttestationMessage` on the wire ships a validator-specific signature verified against that validator's pubkey. Each runner broadcasts its partial signature in its own `PartialSignatureMessages` container with `Type = PTCAttesterPartialSig` (the runner role `RolePTCAttester` is the dispatch discriminator), one single-validator container per PTC-assigned validator, the same per-validator container shape as `ValidatorRegistration` today. Each operator accumulates peers' partial signatures over its own frozen root; when signatures over identical `PayloadAttestationData` reach the reconstruction threshold, it BLS-aggregates and submits one `PayloadAttestationMessage(validator_index, data, signature)` per validator to the beacon node, inside the [75%, 100%] window. Operators on a minority observation never reach threshold and contribute nothing, a non-slashable silent miss (see Security Considerations). The cluster therefore emits, per validator, the observation a threshold of its operators converged on, rather than a single leader's.

False votes and missed votes are equivalent in the payload-extension tally (`should_extend_payload` counts only `True` votes toward the threshold), but not for the next proposer's parent choice: explicit `False` majorities on either field steer the proposer off the full parent in `should_build_on_full` (via `payload_timeliness(..., timely=False)` and `payload_data_availability(..., available=False)`), weight a missed vote does not carry.

There is no QBFT and therefore no value check. PTC attestations are not in the beacon chain slashing predicate, so no slashability call is required.

This SIP adds a new beacon role `BNRolePTCAttester`, a matching runner role `RolePTCAttester`, and a new `PartialSigMsgType` `PTCAttesterPartialSig`.

```go
// types/beacon_types.go additions
var (
    // ... existing values ...
    DomainPTCAttester = [4]byte{0x0C, 0x00, 0x00, 0x00}
)

const (
    // ... existing values ...
    BNRolePTCAttester BeaconRole = 7
)

// types/runner_role.go additions 
const (
    // ... existing values ...
    RolePTCAttester RunnerRole = 7
)

// types/partial_sig_message.go additions
const (
    // ... existing values ...
    PTCAttesterPartialSig PartialSigMsgType = 7
)
```

`RunnerRole` values `1` and `3` are reserved for backward-compat decoding of pre-consolidation messages.

`MapDutyToRunnerRole()` must map `BNRolePTCAttester` to `RolePTCAttester`. PTC reuses the existing `ValidatorDuty`, with one runner instance scoped per PTC-assigned validator (keyed by validator pubkey), the same validator-scoped shape as `ProposerPreferences` ([§5](#5-proposer-preferences-duty)). A slot's PTC is `PTC_SIZE` (512) seats selected from that slot's beacon committees, so a cluster typically holds zero or one PTC seat in a given slot; per-validator scoping reuses the single-validator signing path, and committee-style bundling would save almost nothing.

### 4. Modified Proposer Duty

Relevant consensus-spec references:

- [Validator block and sidecar proposal flow](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#block-and-sidecar-proposal)
- [Gloas `BeaconBlockBody` progressive container](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/beacon-chain.md#beaconblockbody)

Under Gloas, `produceBlockV4` (`POST /eth/v4/validator/blocks/{slot}`, merged via [beacon-APIs #580](https://github.com/ethereum/beacon-APIs/pull/580) and updated by [#624](https://github.com/ethereum/beacon-APIs/pull/624) and [#630](https://github.com/ethereum/beacon-APIs/pull/630)) replaces the pre-Gloas proposer flow; blinded blocks are removed. The request body is a required [`Gloas.BuilderConfig`](https://github.com/ethereum/beacon-APIs/blob/159622d983a703eb03a8a37bb1edeab7ffc3b6bc/types/gloas/builder_entry.yaml) (`min_bid`, `builder_boost_factor`, `builders: List[BuilderEntry, MAX_BUILDER_ENTRIES]`): a missing or undecodable body fails the whole request, while every other failure is per entry, so one bad entry never costs the proposer its slot. Each `BuilderEntry` names a builder `url`, carries the [§5](#5-proposer-preferences-duty)-reconstructed `SignedBuilderRequestAuth` for the slot, and sets per-entry economics; entries never reach a builder, and their `max_execution_payment` is a local bid-valuation cap (the forwarded, builder-binding variant is in [§5](#5-proposer-preferences-duty)). The top-level `min_bid` and `builder_boost_factor` govern p2p bids; an empty `builders` list requests no builder-API bids, leaving p2p bids and self-build. With the required `include_payload` query parameter `true` (mandated below), the beacon node returns `Gloas.BlockContents` on a self-build response and `Gloas.BeaconBlock` on an external-build response. The variant is signaled by the required `execution_payload_included` response field and `Eth-Execution-Payload-Included` header.

*Note (non-normative).* When to issue the `produceBlockV4` request is the residual proposer timing discretion: a later request lets the beacon node observe more bids and can return a higher-value block. The window is tighter for an SSV cluster than for a solo proposer, since the returned block must still clear QBFT, post-consensus signing, and propagation before the earlier attestation deadline ([§1](#1-slot-timing-changes)), so the request must leave room for them. Operator policy, not a protocol requirement.

`ProposerConsensusData` keeps its struct shape (`Duty`, `Version`, `DataSSZ []byte`). At Gloas slots `DataSSZ` carries the SSZ-encoded container

```go
type GloasProposalData struct {
    Block       gloas.BeaconBlock
    PayloadRoot phase0.Root // hash_tree_root(envelope.payload) of the proposer's own self-build envelope; zero for an external bid
}
```

rather than the bare block.

Although the struct shape is unchanged, [`ProposerConsensusData.GetBlockData()`](https://github.com/ssvlabs/ssv-spec/blob/de34c611ce76bf6a40c3f94be0a44bde44d4a7ae/types/consensus_data.go#L192-L253)'s per-version switch (Capella → Fulu today) needs a new `DataVersionGloas` arm that unmarshals `DataSSZ` as `GloasProposalData` and returns its `Block`.

The decided value's `Version` selects how `DataSSZ` is decoded (`GloasProposalData` when `Version >= DataVersionGloas`), and `Version` is leader-supplied. An operator MUST reject any decided value whose `Version` does not equal the fork scheduled at `duty.Slot`. Honest proposers always stamp `Version == fork(duty.Slot)`, so this rejects no honest value.

**Holding `PayloadRoot`.** Operators MUST call `produceBlockV4` with `include_payload=true` at Gloas slots (any operator can lead a later round): a self-build response is then `BlockContents`, so the operator holds its own `payload_root` from the produce call onward and no beacon-node call sits between produce and proposal. A justified value is re-proposed unchanged, `PayloadRoot` included, whatever the re-proposing leader's own bid or payload was.

**Value check.** `ProposerValueCheckF` at Gloas slots additionally requires `PayloadRoot == 0` iff `block.body.signed_execution_payload_bid.message.builder_index != BUILDER_INDEX_SELF_BUILD`, and rejects the value otherwise; an honest leader never trips it. Beyond presence, `PayloadRoot` is leader-trusted (see Security Considerations).

Pre-consensus RANDAO flow is unchanged.

**Post-consensus.** Each operator broadcasts one `PostConsensusPartialSig` packet per round. Its entries are:

- the block root, signed under `DOMAIN_BEACON_PROPOSER`, always; and
- when the decided bid is self-build, the [§6](#6-envelope-signing-self-build-path) blinded envelope root, signed under `DOMAIN_BEACON_BUILDER`, which the operator MUST include.

Entry order is not significant. Receivers derive the expected roots from the decided value, match entries by signing root, and verify each under its own domain: the block root MUST be present, no other root is counted, and at most one share per operator counts per root. Receivers MUST NOT require the envelope entry, so a faulty sender costs itself the envelope share and never the block share; whether a packet carrying an unexpected root is dropped whole or only that entry ignored is an implementation choice. A block-only packet is therefore final for that round; the operator is one share short for the envelope only. Expected-root matching happens in the duty runner, so a packet that arrives before the recipient's own decision is retried rather than dropped. Each root reconstructs independently once it reaches threshold. The signed block is published as `Gloas.SignedBeaconBlock` via the existing block-publish endpoint (`POST /eth/v2/beacon/blocks`); envelope reconstruction and publication are in [§6](#6-envelope-signing-self-build-path). Deployment note: operators MUST decode `GloasProposalData` and accept the two-entry packet before the fork; both are selected by the slot's fork, so pre-Gloas encodings are unchanged.

**`Eth-Builder-Url` echo.** When a builder-API bid wins, the `produceBlockV4` response carries an `Eth-Builder-Url` header (absent for a self-built block or a p2p-bid win); echoed on block publish, it lets the receiving beacon node forward the signed block directly to the winning builder, which releases its payload without waiting for gossip ([beacon-APIs #630](https://github.com/ethereum/beacon-APIs/pull/630)). Within a cluster, each operator's `produceBlockV4` response describes its own candidate block. An operator SHOULD include `Eth-Builder-Url` on publish only when the block in its own `produceBlockV4` response has the decided block root, echoing that response's header byte-for-byte, and SHOULD NOT attach a header taken from a response for a different block: a mismatched echo forwards the signed block to a builder that did not win, a tolerated waste (the builder holds no winning bid for it) rather than a fault. When no publish carries the header (the producing operator does not publish after a round change), the builder still observes the signed block over gossip; the direct forward is a latency optimization, not a liveness requirement.

EIP-7688 does change how that block root is derived. The decided `DataSSZ` bytes are unchanged, but the Gloas block root commits to a progressive `BeaconBlockBody` and its progressive nested types and lists. Operators MUST decode the decided value as `GloasProposalData` and hash its `Block` as `Gloas.BeaconBlock`; positional container merkleization is invalid at Gloas slots.


### 5. Proposer Preferences Duty

Relevant consensus-spec references:

- [Broadcasting SignedProposerPreferences](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/validator.md#broadcasting-signedproposerpreferences)
- [`SignedProposerPreferences` container](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-proposerpreferences)
- [`proposer_preferences` gossip topic](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-proposer_preferences)
- [`execution_payload_bid` gossip validation](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-execution_payload_bid)

Under Gloas, each proposer broadcasts `SignedProposerPreferences` on the `proposer_preferences` p2p topic for future proposal slots within the proposer lookahead (the current epoch up to `MIN_SEED_LOOKAHEAD` epochs ahead). The signed `ProposerPreferences` carries `dependent_root`, `proposal_slot`, `validator_index`, `fee_recipient`, and `target_gas_limit`. `dependent_root` pins the proposer-lookahead epoch's seed via `get_shuffling_dependent_root(store, root, epoch)`; operators populate it from the `dependent_root` returned by [`GET /eth/v2/validator/duties/proposer/{epoch}`](https://github.com/ethereum/beacon-APIs/blob/v5.0.0-alpha.2/apis/validator/duties/proposer.v2.yaml) for the proposal-slot's epoch. This sourcing is enforced at REJECT severity: [`proposer_preferences` gossip validation](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-proposer_preferences) requires the `dependent_root` block to precede the lookahead epoch's start slot (for a proposal in epoch `N`, the block sealing the shuffling as of the end of epoch `N - MIN_SEED_LOOKAHEAD - 1`, which is exactly the duties-endpoint value). A preference stamped with a more recent root, such as the emitting node's current head, is REJECT'd by every peer, so it never reaches builders and the publishing BN takes gossip scoring penalties, unlike a merely stale-fork dependent root, which is IGNORE'd. Builders listen to this topic and use a proposer's preferences to construct `execution_payload_bid` objects for that proposer's slots. This replaces the pre-Gloas out-of-band relay-registration mechanism, which is gone along with blinded blocks; the `ValidatorRegistration` duty is deprecated accordingly (end of this section).

Gossip enforces the handshake at the `execution_payload_bid` topic: each bid requires a matching `SignedProposerPreferences` for its `(proposal_slot, dependent_root)` (otherwise IGNORE'd, not forwarded). The bid `fee_recipient` must match the preference, and the bid `gas_limit` must be EIP-1559-compatible with the proposer's `target_gas_limit` via `is_gas_limit_target_compatible`; both mismatches are IGNORE'd rather than REJECT'd, because preferences are not equivocation-checked, so a mismatch is not provably the bidder's fault. Without this duty broadcast, bids for the validator's slots don't propagate across the network, leaving the BN with no trustless builder options to return.

The flow matches the existing `ValidatorRegistration` / `VoluntaryExit` shape: validator-scoped, non-QBFT, one round of partial signatures, reconstruct, submit. Each operator signs `ProposerPreferences` under `DOMAIN_PROPOSER_PREFERENCES` with the validator's BLS share key; the domain epoch is `compute_epoch_at_slot(proposal_slot)` per the spec's `get_signed_proposer_preferences`, so signatures for the pre-fork emission required below are computed under the Gloas fork domain even while the chain is still on the prior fork. `fee_recipient` is configured cluster-side (cluster-consistent in practice); `target_gas_limit` is operator-configured, with a client default when unset; operators in a cluster must agree byte-for-byte at signing time; `dependent_root` is observed per-operator from the BN's v2 proposer-duties endpoint. Each operator's choice of these three inputs determines its signing root, and operators validate incoming partial signatures against their own derived root (the existing `ValidatorRegistration` expected-root validation); divergence splits signing roots, and reconstruction succeeds only when one root reaches threshold. `fee_recipient` is cluster-consistent in practice; the practical divergence risks are `target_gas_limit` (operator config) and `dependent_root` (observation timing around reorgs and epoch boundaries). The reconstructed `SignedProposerPreferences` is submitted via `POST /eth/v1/validator/proposer_preferences` ([beacon-APIs #608](https://github.com/ethereum/beacon-APIs/pull/608)), which stores it and broadcasts it to the `proposer_preferences` topic.

Trigger: at each epoch boundary, and on duty-dependent-root changes for any epoch in the proposer lookahead, iterate local validators and emit one duty per slot returned by `get_upcoming_proposal_slots(state, validator_index)`. In the `MIN_SEED_LOOKAHEAD` epochs immediately before `GLOAS_FORK_EPOCH`, this SIP requires operators to emit preferences for any local-validator proposal slots in the first Gloas epoch. Emission during the first Gloas epoch itself would also be gossip-valid for slots later in that epoch, but slots early in the epoch leave effectively no post-fork emission time, and builders need a validator's preference before they can construct and gossip bids for its slot; pre-fork emission covers both, aligning with the spec's *"Proposers SHOULD broadcast their preferences in the epoch before the fork"* recommendation in `p2p-interface.md`. The `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot)` key (`validator_index` was dropped from the key upstream; the proposer check already determines it from the other two fields); emission-timing implications are covered in Security Considerations. If the proposer lookahead for an epoch changes, or `dependent_root` changes for an epoch already in the lookahead, cached duties for that epoch are replaced and a new preference is emitted. Because `dependent_root` is part of the gossip key above, the new preference is a distinct key rather than a replacement of the prior one. After a validator-set change (local validators registering, shifting indices), emitters SHOULD additionally delay (re-)emission by a couple of slots so committee peers converge on the new set before the one-shot partials go out; [§7](#7-ssv-message-validation) specifies the matching receive-side tolerance that keeps honest partials valid during the convergence window.

The duty's slot, carried in the `PartialSignatureMessages` envelope, is `proposal_slot` itself: one runner per proposal slot, matching ssv-spec's requirement that a partial-signature message's slot equal its duty's slot (`validatePartialSigMsgForSlot`). At emission time that slot is up to the proposer lookahead in the future. A changed preference for the same slot carries a new signing root, while a delivery retry can repeat one. These cases need dedicated message-validation rules, specified in [§7](#7-ssv-message-validation).

This SIP adds a new beacon role `BNRoleProposerPreferences`, a matching runner role `RoleProposerPreferences`, and a new `PartialSigMsgType` `ProposerPreferencesPartialSig`.

```go
// types/beacon_types.go additions
var (
    // ... existing values ...
    DomainProposerPreferences = [4]byte{0x0D, 0x00, 0x00, 0x00}
)

const (
    // ... existing values
    BNRoleProposerPreferences BeaconRole = 8
)

// types/runner_role.go additions
const (
    // ... existing values
    RoleProposerPreferences RunnerRole = 8
)

// types/partial_sig_message.go additions
const (
    // ... existing values
    ProposerPreferencesPartialSig PartialSigMsgType = 8
)
```

`MapDutyToRunnerRole()` must map `BNRoleProposerPreferences` to `RoleProposerPreferences`.

#### Deprecation of the `ValidatorRegistration` duty

Gloas removes every protocol consumer of `SignedValidatorRegistrationV1`: builders source `fee_recipient` and `target_gas_limit` from the `proposer_preferences` topic instead, the relay path that consumed registrations is gone with blinded blocks ([§4](#4-modified-proposer-duty)), and the Gloas builder-API workstream itself deprecates `ValidatorRegistrationV1` in favor of `ProposerPreferences` ([builder-specs PR #138](https://github.com/ethereum/builder-specs/pull/138), merged 2026-06-11; see also [builder-specs issue #150](https://github.com/ethereum/builder-specs/issues/150)), with the registration's authentication function surviving as the out-of-protocol `SignedBuilderRequestAuth` ([builder-specs #165](https://github.com/ethereum/builder-specs/pull/165), merged 2026-08-24). This SIP therefore deprecates the duty at the fork: operators emit no `BNRoleValidatorRegistration` duties for epochs at or after `GLOAS_FORK_EPOCH`, message validation treats `ValidatorRegistrationPartialSig` messages for Gloas-or-later slots as invalid, and pre-Gloas slots are unchanged. `BNRoleValidatorRegistration`, `RoleValidatorRegistration`, and `ValidatorRegistrationPartialSig` keep their numeric values, reserved for pre-Gloas operation and backward-compat decoding, the same treatment as `RunnerRole` values `1` and `3` ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)). Where a beacon node sources self-build `fee_recipient` / `target_gas_limit` after the fork is BN configuration outside the SSV protocol, and it needs no distributed signature: registration signatures existed to authenticate proposers to untrusted relays (`prepare_beacon_proposer` carries only `fee_recipient` today). The Gloas analog of that authentication, for clusters that opt into direct builder connections, is the distributed `BuilderRequestAuth` signature specified next.

#### Builder request auth extension

Relevant references:

- [`DOMAIN_BUILDER_REQUEST_AUTH` constant](https://github.com/ethereum/builder-specs/blob/5aef563dc3532a5009fef02bae97ca563ec28e5b/specs/gloas/builder.md#constants) and [signing functions](https://github.com/ethereum/builder-specs/blob/5aef563dc3532a5009fef02bae97ca563ec28e5b/specs/gloas/builder.md#signing)
- [Constructing the `BuilderRequestAuth`](https://github.com/ethereum/builder-specs/blob/5aef563dc3532a5009fef02bae97ca563ec28e5b/specs/gloas/validator.md#constructing-the-builderrequestauth)
- [`submitBuilderPreferences` beacon endpoint](https://github.com/ethereum/beacon-APIs/blob/159622d983a703eb03a8a37bb1edeab7ffc3b6bc/apis/validator/builder_preferences.yaml)
- [keymanager-APIs #88](https://github.com/ethereum/keymanager-APIs/pull/88) (informative: per-validator builder-config vocabulary)

Direct builder connections ([§4](#4-modified-proposer-duty)) authenticate each request with `SignedBuilderRequestAuth`: a validator-key BLS signature over `BuilderRequestAuth { data: ByteList[4096], slot }` under `DOMAIN_BUILDER_REQUEST_AUTH` (`0x0B000001`), computed with `compute_domain` defaults (genesis fork version, zeroed `genesis_validators_root`), the same chain-independent construction as the deprecated `ValidatorRegistrationV1` and one byte from [§6](#6-envelope-signing-self-build-path)'s `DOMAIN_BEACON_BUILDER` (`0x0B000000`). `slot` is the proposal slot the request is authorized for, never the slot at which it is signed or sent, so it is a pure function of the proposer lookahead. `data` is the token agreed with the builder out of band, signed exactly as configured with no canonicalization at any hop; when nothing is agreed it defaults to the UTF-8 bytes of the builder's advertised URL, and zero-length `data` is invalid. One signature therefore covers a `(data, proposal_slot)` pair, and entries sharing `data` share it.

The signing round rides the proposer-preferences duty: same runner and role, same triggers, same `proposal_slot` message slot, with partial signatures typed `RequestAuthPartialSig` instead of `ProposerPreferencesPartialSig`. No new beacon role, runner role, or `MapDutyToRunnerRole` entry is added. For each configured builder entry of an upcoming proposal slot, each operator derives `BuilderRequestAuth{data, proposal_slot}`, signs it with the validator's BLS share key, and broadcasts one single-entry `PartialSignatureMessages` container per distinct auth signing root. Operators validate incoming partials against the root set derived from their own configured entries and collect per root, independently of the preference round and of other roots; a root reaching the reconstruction threshold yields one `SignedBuilderRequestAuth`, held for the proposal slot. Auth roots contain no `dependent_root`, so the re-emission triggers above reproduce byte-identical roots: an operator broadcasts each root's partial once per emission (an honest retry may repeat it; [§7](#7-ssv-message-validation) treats repeats as duplicates), and collection continues until the proposal slot. The preference flow is unchanged, and neither round gates the other. Deployment note: the `RequestAuthPartialSig` wire variant must be decodable network-wide before any operator emits it, because a container with an unknown partial-signature type fails structural decoding and is REJECT'd, penalizing the forwarding peer, unlike the IGNORE-classed budget rules of [§7](#7-ssv-message-validation).

```go
// types/beacon_types.go additions
var (
    // ... existing values ...
    DomainBuilderRequestAuth = [4]byte{0x0B, 0x00, 0x00, 0x01}
)

// types/partial_sig_message.go additions
const (
    // ... existing values ...
    RequestAuthPartialSig PartialSigMsgType = 9
)
```

Builder entries are cluster configuration and MUST be byte-identical across all `n` operators, `data` included, keeping the auth path `f`-fault-tolerant; divergence consequences are in Security Considerations. SSV caps configured entries at `8` per validator, a sub-cap of the beacon-API's `MAX_BUILDER_ENTRIES` (64) that bounds the [§7](#7-ssv-message-validation) budget.

The reconstructed signature feeds both channels through the operator's own beacon node: as each entry's `auth` in the [§4](#4-modified-proposer-duty) produce body, and optionally ahead of time (an upstream MAY, submitted best-effort) via `POST /eth/v1/validator/builder_preferences`, which forwards each `BuilderPreferencesEntry` (`proposer_pubkey`, `url`, `auth`, `max_execution_payment`) to its builder byte-for-byte; this forwarded `max_execution_payment` binds the builder's bids for the slot, unlike the [§4](#4-modified-proposer-duty) valuation cap. A builder whose root has not reached threshold by the proposal slot is omitted from the produce body; p2p bids and self-build are unaffected, so the extension is opt-in and never on the proposal critical path.

In the `MIN_SEED_LOOKAHEAD` epochs immediately before `GLOAS_FORK_EPOCH`, operators with configured entries run the auth round for local-validator proposal slots in the first Gloas epoch, alongside the pre-fork preference emission required above. The domain is chain-independent, so pre-fork signing needs no fork-domain special case, and the [§7](#7-ssv-message-validation) fork gate passes because message slots are post-fork proposal slots by construction.

### 6. Envelope Signing (Self-Build Path)

Relevant consensus-spec references:

- [`ExecutionPayloadEnvelope` container](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/beacon-chain.md#executionpayloadenvelope)
- [EIP-7495 `ProgressiveContainer` merkleization](https://eips.ethereum.org/EIPS/eip-7495#merkleization)
- [`execution_payload` gossip topic (carries `SignedExecutionPayloadEnvelope`)](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-execution_payload)
- [`POST /eth/v1/beacon/execution_payload_envelopes` endpoint](https://github.com/ethereum/beacon-APIs/pull/624)

On the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD` per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)), the proposer signs `SignedExecutionPayloadEnvelope` alongside the block. The [§4](#4-modified-proposer-duty) decision pins exactly one valid envelope: `bid.block_hash` commits to the whole payload, `bid.execution_requests_root` commits to the requests, and [`verify_execution_payload_envelope`](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/fork-choice.md#new-verify_execution_payload_envelope) checks every envelope field against the committed bid and block. With `PayloadRoot` in the decided value, every field of the envelope's signing input is derivable from that value, so the envelope signature rides the [§4](#4-modified-proposer-duty) post-consensus packet.

#### Blinded envelope type

The signing root is computed over a blinded form that substitutes `payload` with `payload_root: Root = hash_tree_root(payload)` and `execution_requests` with `execution_requests_root: Root = hash_tree_root(execution_requests)`. The blinded type MUST use the same five-field `ProgressiveContainer` shape as `ExecutionPayloadEnvelope` (serialization is unaffected; only merkleization differs). Substituting a field with its root preserves the progressive container root, so `hash_tree_root(BlindedExecutionPayloadEnvelope) == hash_tree_root(ExecutionPayloadEnvelope)`, and a BLS signature over the blinded signing root is valid for the full envelope.

```go
// SSZ type: ProgressiveContainer(active_fields=[1, 1, 1, 1, 1])
type BlindedExecutionPayloadEnvelope struct {
    PayloadRoot           phase0.Root // == hash_tree_root(envelope.payload)
    ExecutionRequestsRoot phase0.Root // == hash_tree_root(envelope.execution_requests)
    BuilderIndex          uint64
    BeaconBlockRoot       phase0.Root
    ParentBeaconBlockRoot phase0.Root
}
```

Its root is:

```text
mix_in_active_fields(
    merkleize_progressive([
        payload_root,
        execution_requests_root,
        hash_tree_root(builder_index),
        hash_tree_root(beacon_block_root),
        hash_tree_root(parent_beacon_block_root),
    ]),
    [1, 1, 1, 1, 1],
)
```

`payload_root` MUST be the progressive Gloas `ExecutionPayload` root, and `execution_requests_root` MUST be the progressive Gloas `ExecutionRequests` root, whose request lists (including [EIP-8282](https://eips.ethereum.org/EIPS/eip-8282)'s builder deposits and exits) are `ProgressiveList`s that mix in no list maximum. Blinded-to-full root equivalence MUST be tested against a fixed expected root, not only by comparing the two locally derived values: a same-implementation comparison passes even when both sides share the same incorrect merkleization.

#### Derivation from the decided value

Every field comes from the [§4](#4-modified-proposer-duty)-decided `GloasProposalData`:

- `PayloadRoot` is the decided `PayloadRoot`;
- `ExecutionRequestsRoot` is `block.body.signed_execution_payload_bid.message.execution_requests_root`;
- `BuilderIndex` is `BUILDER_INDEX_SELF_BUILD`;
- `BeaconBlockRoot` is `hash_tree_root(block)`;
- `ParentBeaconBlockRoot` is `block.parent_root`.

The signing root is the blinded container's progressive root under `DOMAIN_BEACON_BUILDER` (`0x0B000000`), domain epoch = `compute_epoch_at_slot(duty.slot)` (matching `verify_execution_payload_envelope_signature`'s `get_domain(state, DOMAIN_BEACON_BUILDER)` against the proposal-slot state); by root equivalence this is the full envelope's signing root. An operator computes it from the decided value alone, with no beacon-node call, and signs it as the envelope entry of its post-consensus packet ([§4](#4-modified-proposer-duty)).

#### Publication

All operators reconstruct the envelope signature from the same packets that carry the block shares. Only an operator whose beacon node built the decided block holds the payload bytes behind `PayloadRoot`, so only it can form a publishable body. After the [§4](#4-modified-proposer-duty) decision, an operator is the builder operator iff the block in its own produce response has the decided block root (the same local test [§4](#4-modified-proposer-duty) uses for the `Eth-Builder-Url` echo) and its own full envelope blinds to the derived blinded envelope in every field. Operators sharing a beacon node may both match; they publish identical content, which is harmless. Every other operator publishes nothing.

Publication follows [beacon-APIs #624](https://github.com/ethereum/beacon-APIs/pull/624): `Eth-Blob-Data-Included: false` carries `SignedExecutionPayloadEnvelope`, while `true` carries `SignedExecutionPayloadEnvelopeContents`. The builder operator SHOULD publish the `Contents` form to each of its beacon nodes, giving the reveal the same all-node redundancy as the [§4](#4-modified-proposer-duty) block publish; the `false` form is accepted only by the node that produced the block. A beacon node validates an envelope against the block it belongs to and ignores one whose block it has not seen, so the builder operator SHOULD publish to nodes that have accepted the block, or retry until they have. No public `SignedBlindedExecutionPayloadEnvelope` is used. The execution payload, execution requests, blobs, and KZG proofs never cross the SSV wire; only the payload and requests roots do.

#### Timing

The signed envelope must reach PTC beacon nodes before `get_payload_due_ms()` (the `PAYLOAD_DUE_BPS` cutoff, 50% of the slot; [§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)), with margin for gossip. Because the envelope shares travel in the block's post-consensus packet, the envelope signature is normally reconstructable at the same moment as the block signature, before the attestation deadline ([§1](#1-slot-timing-changes), 25%); the remaining budget covers the block publish, the envelope publish, and gossip. An envelope that misses the cutoff is the bounded degradation in Security Considerations.

### 7. SSV Message Validation

SSV pubsub message validation is content-agnostic: it never validates beacon-object content, so it enforces only structure and metadata. This section pins those rules for the two new roles, the Gloas proposer post-consensus packet, and the deprecated role; all structural, signature, and topic rules for existing roles apply unchanged. Classifications follow the existing convention: REJECT penalizes the sending peer while IGNORE drops without forwarding.

Both new roles are validator-scoped, like `RoleValidatorRegistration` and `RoleVoluntaryExit`: the `MessageID` carries the validator public key, routing reuses the cluster's existing subnet (no new topic), and each `PartialSignatureMessages` container carries at most one entry (the non-committee packet rule, REJECT above 1). Partial-signature type must be one the role admits (REJECT otherwise): one type per role, except `RoleProposerPreferences`, which also admits `RequestAuthPartialSig` ([§5](#5-proposer-preferences-duty)'s builder-auth extension) under its own budget below. Consensus messages are REJECT'd for both non-QBFT roles, joining the existing registration/exit rule. The existing `RoleProposer` gains one structural exception at Gloas slots, derived below.

| Role (wire) | Consensus messages | Partial-sig type; count per (`MessageID`, signer, slot) | Message slot | Earliness allowance | Lateness TTL | Duty assignment check | Duty limit per epoch |
|---|---|---|---|---|---|---|---|
| `RolePTCAttester` (7) | REJECT | `PTCAttesterPartialSig` (7); 1 | PTC attestation slot | none | 3 slots | PTC assignment at slot (IGNORE) | 2 (IGNORE) |
| `RoleProposerPreferences` (8) | REJECT | `ProposerPreferencesPartialSig` (8); up to 4 distinct signing roots. `RequestAuthPartialSig` (9); up to 8 distinct signing roots | `proposal_slot` | `(1 + MIN_SEED_LOOKAHEAD) * SLOTS_PER_EPOCH` slots | 2 slots | proposer assignment at `proposal_slot` (IGNORE) | `SLOTS_PER_EPOCH` (IGNORE) |

Lateness TTL uses the existing deadline convention: a message is late once received after `slot_start(slot + TTL)` plus the clock-error margin. Duty limits count distinct duty slots per (`MessageID`, signer, epoch of the message slot). Duty assignment checks apply only once the local node knows the duties for the message slot's epoch (a not-yet-fetched epoch is tolerated rather than IGNORE'd, since the message can legitimately arrive first).

Validation state backing these rules (recorded signing roots, duty counts) MUST be retained while any message it gates remains acceptable: state for message slot `S` lives from the earliest acceptable arrival (`S` minus the earliness allowance) through `S`'s lateness deadline. For `RoleProposerPreferences` that is earliness + lateness = `(1 + MIN_SEED_LOOKAHEAD) * SLOTS_PER_EPOCH + 2` slots, and acceptable message slots span up to four consecutive epochs at any instant, so duty counts MUST be kept for each of those epochs; for the other roles the window is just the lateness TTL, already met in practice. Evicting live state silently re-opens the budgets above: a previously recorded root would be accepted and re-forwarded as a new distinct root.

A view that predates the node's most recent validator-set change, or its most recently detected dependent-root change for that epoch, is treated the same way: skip the check, don't IGNORE, until a refresh completed after that change is installed. Detection is whatever the client's duty-refresh mechanism provides: a client that re-fetches and wholesale-replaces the view each slot detects and installs in one step, leaving no skip window, while an event-driven client skips between the event and the refetch landing. This is load-bearing for the one-shot partial-signature checks across these roles: a validator-index or dependent-root change can cause an honest message to be emitted while invalidating a peer's cached view. Gossip's seen-cache can suppress an immediate re-broadcast of a deterministic partial. A wrongly-dropped first copy can therefore starve a short-lived duty or delay reconstruction until a later retry. Anti-spam stays bounded by committee membership, the distinct-signing-root cap, and the per-epoch duty-count cap.

Derivations, per row:

- **PTC (7).** One partial-signature round, no pre/post split: `PTCAttesterPartialSig` is the only type, once per duty. The vote is consumed within its own slot and included at slot + 1 ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)), so the short non-committee TTL applies. The duty limit follows from `compute_ptc` drawing members exclusively from `get_beacon_committee(state, slot, i)`: a validator belongs to exactly one slot's beacon committees per epoch, so the honest bound is one PTC duty per validator per epoch, plus the +1 reorg margin the existing registration and aggregation limits already use. Multiple PTC seats within one slot are still one duty: the validator signs one `PayloadAttestationData`, and seat multiplicity lives in the aggregation bits.
- **ProposerPreferences (8).**
  - *Earliness:* the message slot is `proposal_slot` ([§5](#5-proposer-preferences-duty)), up to the proposer lookahead in the future at emission, hence the allowance.
  - *Lateness:* the tight TTL drops stale preferences as replays.
  - *Slot-advance exempt:* a signer holds its whole lookahead at once, so a lower-slot message is a concurrent duty, not a stale one (lateness is its past bound).
  - *Dedup:* The base one-per-(signer, slot) rule gains a bounded exception for distinct signing roots at one proposal slot. Each of the first four distinct roots per (`MessageID`, signer, `proposal_slot`) is eligible for ACCEPT, subject to all other validation rules. When a message with a new root receives ACCEPT, the implementation MUST record the root. It MUST NOT record roots from messages that receive IGNORE or REJECT. If this dedup rule is the only failing condition, implementations MUST classify the message as IGNORE in either case:
    - The root was recorded previously, regardless of the propagation peer.
    - The root is new, but four roots were recorded previously for the same tuple.

    Preference-input changes can produce distinct roots, notably after a `dependent_root` reorg ([§5](#5-proposer-preferences-duty)). The cap is policy headroom. An honest sender retry or restart can repeat an earlier root after the recipient's gossip duplicate cache expires. Therefore, repetition does not prove peer fault.
  - *Request-auth dedup:* `RequestAuthPartialSig` messages follow the same two IGNORE cases and record-on-ACCEPT discipline, with a separate budget of 8 distinct roots per (`MessageID`, signer, `proposal_slot`), one per configured builder entry ([§5](#5-proposer-preferences-duty) caps entries at 8; entries sharing `data` share a root). Budgets are tracked per partial-signature type; neither consumes the other. The IGNORE rationale is strictly stronger here: honest re-triggers reproduce identical roots by design ([§5](#5-proposer-preferences-duty)). Type-9 messages ride existing duty slots and add no distinct slots to the per-epoch duty limit.
  - *Duty limit:* at most one proposal per slot.
- **Proposer (existing role, Gloas post-consensus packet).** At slots whose fork is Gloas or later, a `PostConsensusPartialSig` packet for `RoleProposer` carries up to two entries (REJECT above two), and every entry MUST carry the same signer and validator index (REJECT otherwise). The exception is selected by role and message slot, not by build path, since validation cannot see whether the decided bid is self-build. The existing one-post-consensus-packet-per-round count is unchanged, and pre-consensus (`RandaoPartialSig`) packets keep the one-entry rule. Which roots the entries carry is a [§4](#4-modified-proposer-duty) runner check.

Epoch-aware fork gates, a new rule class. Both are REJECT because they are slot-scoped rather than wall-clock-scoped: the gated condition is carried in the message itself, so no honest ordering race can produce it.

| Rule | Classification | Explanation |
|---|---|---|
| Role in {7, 8} and `epoch(msg.slot) < GLOAS_FORK_EPOCH` | REJECT | These duties do not exist before the fork. [§5](#5-proposer-preferences-duty)'s required pre-fork emission carries post-fork `proposal_slot` values by construction, so it passes this gate with no special case. |
| `RoleValidatorRegistration` (4) and `epoch(msg.slot) >= GLOAS_FORK_EPOCH` | REJECT | [§5](#5-proposer-preferences-duty) deprecates the duty at the fork; honest registrations are always stamped with pre-fork slots. Pre-Gloas slots remain valid and unchanged. |

## Security Considerations

### Mixed merkleization splits threshold signatures

Operators using positional and progressive implementations can hold identical value bytes (a QBFT-decided `DataSSZ`, or the [§6](#6-envelope-signing-self-build-path) blinded envelope derived from it) while deriving different signing roots. Their partial signatures do not reconstruct together. If a threshold uses the obsolete positional root, it can reconstruct a signature that the beacon network rejects. Fork-aware Gloas types and cross-implementation root fixtures are therefore consensus-critical interoperability requirements, not encoding-only upgrades.

### `GloasBeaconVoteValueCheckF` must include `AttestationDataIndex` in slashability checks

Under Gloas, `AttestationData.Index` is part of the attestation data root and therefore part of the double-vote slashing predicate. `GloasBeaconVoteValueCheckF` must reconstruct the full Gloas `AttestationData` with `Index` from the decided `GloasBeaconVote.AttestationDataIndex` before calling `IsAttestationSlashable`; otherwise an operator could sign `index=0` and `index=1` for the same `(source, target)` in the same slot without the predicate tripping.

### Gloas `AttestationData.Index` is trusted from the QBFT leader

The Gloas `AttestationData.Index` value check ([§2](#2-modified-attestation-duty)) does not require the QBFT-decided value to match each operator's local BN view. Requiring local agreement would fail QBFT rounds whenever operators observe fork-choice state at slightly different times around the deadline, a normal gossip-lag scenario. Accepted tradeoff: a malicious QBFT leader can push a value contrary to the cluster's majority BN observation. This matches existing ssv-spec treatment of `BeaconVote.BlockRoot`, which is trusted from the leader because BNs legitimately diverge on fork-choice head.

### PTC reconstruction is honest-convergence, not consensus

PTC runs no QBFT ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)): each operator signs the `PayloadAttestationData` its own beacon node observed at the 75% broadcast mark, and a per-validator signature reconstructs only when a threshold of operators converged on byte-identical data. There is no leader to push a value contrary to the cluster's observation, and an operator can only ever vote its own honest observation. The cost is liveness rather than safety: when operators' beacon nodes split across observations (envelope-arrival jitter at the 50% `PAYLOAD_DUE_BPS` boundary, head or blob-availability jitter at evaluation time around `MAXIMUM_GOSSIP_CLOCK_DISPARITY`, or operators on diverged forks), no observation may reach threshold and the cluster's vote for that validator is a silent miss. That miss is non-slashable and its only effect is the foregone contribution to the `PTC_SIZE/2` fork-choice tally, bounded by SSV's PTC seat share. The off-slot-root case that Gloas gossip guards with an IGNORE-level block-at-assigned-slot check in [`payload_attestation_message` gossip validation](https://github.com/ethereum/consensus-specs/blob/a5a1bc630401eedbe2f3d87934c99012578c113b/specs/gloas/p2p-interface.md#new-payload_attestation_message) cannot arise here: each operator signs the block it observed for `duty.slot`.

### Config divergence silently disables trustless builder bids

`ProposerPreferences` reconstruction requires a quorum of operators to derive the same signing root, which depends on `target_gas_limit` (per-operator config) and `dependent_root` (per-operator BN observation). Divergence on either splits signing roots; if no root reaches threshold, there is no reconstructed signature, no gossip publication, and therefore no matching preference on the `execution_payload_bid` topic; bids for the slot are IGNORE'd by gossip ([§5](#5-proposer-preferences-duty)), leaving the BN with no trustless builder options to return. Same reconstruction failure shape as `ValidatorRegistration` today.

### Builder `data` divergence silently drops that builder

`BuilderRequestAuth.data` is per-builder configuration, not an observation: a single operator configuring different bytes, even a trailing slash or case difference in a URL the `data` defaults to, splits that builder's signing roots below threshold, and the builder receives no authenticated requests for the validator's slots ([§5](#5-proposer-preferences-duty)). Per-root packets bound the damage to that builder; other entries, p2p bids, and self-build are unaffected. Unlike the trustless-bid entry above, nothing on gossip reveals the failure: proposals keep succeeding through gossiped bids or self-build, so the miss is yield-only and visible only in reconstruction telemetry.

### Unsigned produce-body knobs are operator policy, not consensus

`min_bid`, `builder_boost_factor`, and the produce-body `max_execution_payment` ([§4](#4-modified-proposer-duty)) are never signed and never cross the SSV wire: divergence changes only which candidate block each operator's beacon node returns, and the cluster's proposer value check deliberately validates no bid economics, so the decided block reflects the deciding leader's policy. This is consensus-safe. A future value-check rule that validates bid contents against local policy would convert knob divergence into round-change liveness risk, so such a tightening must be paired with an enforced config-consistency requirement.

### Too-early `SignedProposerPreferences` publication pins the wrong preference

Because the `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot)` key ([§5](#5-proposer-preferences-duty)), reconstructing and publishing a preference before all its inputs are final can durably pin a wrong-input preference: a later corrected message for the same key is dropped by gossip rather than treated as a replacement. In particular, a re-emission that changes only `fee_recipient` or `target_gas_limit` under an unchanged `(dependent_root, proposal_slot)` is IGNORE'd network-wide; only a `dependent_root` change opens a new key. Builders keep using the stale preference, and bids matching the corrected values fail the [§5](#5-proposer-preferences-duty) handshake. Operators must therefore hold publication until `dependent_root`, `fee_recipient`, and `target_gas_limit` are all final for the key, and publish changed preference content only when the key itself changes (notably when `dependent_root` shifts due to reorg, or `proposer_lookahead` reassigns the validator to a different slot). Distinct from the config-divergence entry above: there, divergence prevents publication; here, premature publication pins the wrong preference more durably than no publication would.

### Late `dependent_root` change near the proposal slot may leave the slot with no matching bid

A late `dependent_root` change tightens the re-emission window but does not block it: the changed root forms a new `(dependent_root, proposal_slot)` key, which gossip accepts (first-message pinning binds a fixed key), so the risk here is the remaining time budget, not gossip rejection. Under non-finality, a deep reorg affecting the end-of-p-2 dependent block forces the proposer to re-emit `SignedProposerPreferences` with the new root; if the re-emission + builder-bid gossip round-trip cannot complete before the proposal deadline, the slot falls through to self-build with a compressed produce-and-decide window.

### Builder-operator failure or late publication misses the slot's envelope

The slot's envelope is missed if the builder operator fails to publish, if fewer than a threshold of envelope shares arrive ([§4](#4-modified-proposer-duty)), or if the signed envelope reaches PTC validators after `get_payload_due_ms()`, too late to observe before the cutoff. In each case PTC records `payload_present = FALSE` ([§3](#3-new-duty-payload-timeliness-committee-ptc-attestation)); the proposer forfeits the payload reward. No worse than the no-envelope-signing baseline for the self-build path.

### A wrong `PayloadRoot` costs a self-build reveal, never a wrong payload

`PayloadRoot` is the one field of the [§6](#6-envelope-signing-self-build-path) signing input that operators cannot check against the decided block, so it is leader-trusted ([§4](#4-modified-proposer-duty)). A Byzantine leader can decide a value whose `PayloadRoot` matches no payload; the cluster then threshold-signs an envelope nobody can publish, and the slot's reveal is missed, the bounded degradation above. Safety holds regardless: a publishable envelope needs the payload bytes behind its `PayloadRoot`, finding a payload for a fabricated root is a hash preimage, and the envelope signature (`DOMAIN_BEACON_BUILDER`) appears in no Gloas slashing predicate. A Byzantine non-leader cannot split the cluster: QBFT admits one justified value and nothing else reaches the signing input.

## Open Questions / Upstream Watchlist

This section is intentionally limited to upstream items that could still change the normative SSV behavior described above. If any of these settle differently, this SIP should be updated.

- `produceBlockV4WithBid`: [beacon-APIs #627](https://github.com/ethereum/beacon-APIs/pull/627) (open) would let the validator client fetch bids and hand one to each beacon node, reshaping [§4](#4-modified-proposer-duty)'s produce flow for multi-BN setups.
- Release pin: the [§4](#4-modified-proposer-duty)/[§5](#5-proposer-preferences-duty) builder flow cites unreleased beacon-APIs master ([#630](https://github.com/ethereum/beacon-APIs/pull/630), merged 2026-08-24); watch the next beacon-APIs release for pre-release changes, in particular whether the `BuilderConfig` body stays required.
- Canonical URL byte-encoding for defaulted `data` is unspecified upstream; a canonicalization rule would change the footing of [§5](#5-proposer-preferences-duty)'s byte-identity requirement.
- Builder-side handling of a repeated `submitBuilderPreferences` for the same (proposer, `auth.message.slot`) is unspecified (replace vs first-wins), and there is no beacon-node read-back endpoint to audit what was submitted; this SIP therefore recommends no resubmit-to-correct behavior until either settles.

## Acknowledgements

Thanks to @diegomrsantos, @GalRogozinski, and @iurii-ssv for review, design discussion, and feedback on the Gloas integration, including PTC handling, proposer preferences, and self build envelope signing.
