|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Shane Moore    | ePBS (EIP-7732) Support    | Core       | draft               | 2026-04-18 |

## Summary

Describes the SSV spec changes needed to keep SSV operators performing validator duties correctly after ePBS, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732), is implemented in Ethereum's consensus layer Gloas fork. Based on the pinned [Gloas consensus-spec snapshot](https://github.com/ethereum/consensus-specs/tree/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas) (`ethereum/consensus-specs@4a4937be`, reviewed 2026-05-18).

Validator client related changes via ePBS:
1. earlier slot deadlines
2. attestation handling changes, including preservation of the Gloas `attestation.data.index` semantics
3. the new Payload Timeliness Committee (PTC)
4. the Gloas proposer flow using `produceBlockV4`. The SSV cluster signs only the `Gloas.BeaconBlock`; signing of `SignedExecutionPayloadEnvelope` (the validator-signed envelope used in the self-build path) is intentionally out of scope — see §4.
5. the new `SignedProposerPreferences` message must be submitted if the node operator wants to be able to select block bids received over p2p

## Motivation

Gloas changes validator duties in ways that break a few current SSV assumptions:

- attestation `index` is no longer safely reconstructible from local validator duty data
- the new PTC duty has a late in-slot deadline
- SSV validators must broadcast `SignedProposerPreferences` or they cannot accept external-builder bids for their slots

## Rationale

Key design choices and why:

- **New `GloasBeaconVote` carries `AttestationDataIndex`.** In Gloas, `AttestationData.Index` is BN-supplied and part of the signed attestation root, so it must travel through QBFT consensus data rather than being reconstructed locally. A dedicated Gloas-only type keeps pre-Gloas `BeaconVote` wire bytes unchanged.
- **PTC is a committee-scoped runner.** `PayloadAttestationData` is validator-independent (like `BeaconVote`), while each PTC-assigned validator still needs its own BLS signature and submission object. This matches the existing committee-runner pattern from `committee_consensus.md`.
- **Proposer-preferences is validator-scoped and non-QBFT.** `fee_recipient` already lives per-validator on `Share.FeeRecipientAddress`; `target_gas_limit` lives in operator config (currently `DefaultGasLimit = 30_000_000` in `types/beacon_types.go`, with runtime overrides, same as the existing validator-registration flow). The signed object is therefore agreed off-chain, so there is nothing to reach consensus over. The registration-like one-round partial-sig-and-submit flow from `voluntary_exit.md` fits directly.
- **Block QBFT remains scoped to the `Gloas.BeaconBlock`.** `ProposerConsensusData.data_ssz` carries the block SSZ, matching today's shape. Distributed signing of `SignedExecutionPayloadEnvelope` is out of scope; see §4 for the rationale.

## Specification

### 1. Slot Timing Changes

All existing validator duty deadlines shift earlier in the slot. A new PTC deadline is added.

Relevant consensus-spec references:

- [Validator time parameters](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#time-parameters)

| Duty | Pre-ePBS | Post-ePBS (Gloas) |
|------|----------|--------------------|
| Attestation | 1/3 slot (~4s) | 1/4 slot (25%, ~3s) |
| Sync Committee Message | 1/3 slot | 1/4 slot (25%) |
| Aggregation | 2/3 slot (~8s) | 1/2 slot (50%, ~6s) |
| Sync Committee Contribution | 2/3 slot | 1/2 slot (50%) |
| PTC Attestation | - | 3/4 slot (75%, ~9s) |

### 2. Modified Attestation Duty

Relevant consensus-spec references:

- [Validator attestation changes](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#attestation)

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

A new `GloasBeaconVote` struct mirrors `BeaconVote` plus an `AttestationDataIndex` field, matching the `phase0.CommitteeIndex` (a `uint64` alias) type of `AttestationData.Index` in consensus specs so reconstruction is a direct field assignment. Gloas-era QBFT instances decide on `GloasBeaconVote`; pre-Gloas instances continue to decide on the existing `BeaconVote`, whose SSZ layout is unchanged. A separate type (rather than extending `BeaconVote` in place) is the cleanest way to make pre-Gloas and Gloas wire bytes mutually-rejecting on length mismatch, since SSZ derives do not support fork-conditional fields. The restricted Gloas value space (`0` = `EMPTY`, `1` = `FULL` for non-same-slot attestations; `0` for same-slot) is enforced in the value check below, not at the type level.

After the Gloas fork epoch has activated on all networks and pre-Gloas slots are no longer reachable in normal operation, a follow-up SIP can retire `BeaconVote` and rename `GloasBeaconVote` back to `BeaconVote`.

```go
// Existing (ssv-spec types/consensus_data.go); unchanged
type BeaconVote struct {
    BlockRoot phase0.Root `ssz-size:"32"`
    Source    *phase0.Checkpoint
    Target    *phase0.Checkpoint
}

// New (Gloas only)
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

The [Gloas same-slot rule](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#attestation) (`block.slot == data.slot ⇒ data.index = 0`) is not enforced locally: the cluster has only the QBFT-decided `BlockRoot` and trusts `AttestationDataIndex` from the leader. A single bad same-slot `index=1` is rejected by the ethereum network and ignored on chain but is not slashable, while cross-`index` equivocation over the same `(source, target, slot, BlockRoot)` is still caught by `IsAttestationSlashable` per the previous bullet.

Pre-Gloas slots continue to run `BeaconVoteValueCheckF()` unchanged.

#### Implementation note: aggregation path

The `BNRoleAggregator` duty (handled by the aggregator-committee runner) fetches aggregated attestations from the Beacon API's aggregate-attestation endpoint with `attestation_data_root` as an input. Implementations must compute that root from the BN-supplied `AttestationData` (including its Gloas `index`).

### 3. New Duty: Payload Timeliness Committee (PTC) Attestation

Relevant consensus-spec references:

- [Validator payload timeliness attestation flow](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#payload-timeliness-attestation)
- [Beacon-chain payload attestation containers](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/beacon-chain.md#payloadattestationdata)
- [Fork-choice payload attestation deadline](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/fork-choice.md#new-get_payload_attestation_due_ms)

PTC is a per-slot consensus-layer-selected set of validators that attests to payload and blob availability for the slot's beacon block.

Each validator signs a `PayloadAttestationData` object carrying `beacon_block_root`, `slot`, `payload_present`, and `blob_data_available`, then submits a validator-specific `PayloadAttestationMessage(validator_index, data, signature)` to the beacon node.

At the start of each epoch, SSV should fetch PTC duties for the next epoch and refresh them on duty-dependent-root changes. Because PTC duty responses may be sparse and incomplete, a changed duty-dependent root for an epoch should replace the cached duties for that epoch rather than being merged.

Two coincident 75%-of-slot deadlines bound the runner: `PAYLOAD_ATTESTATION_DUE_BPS = 75%` (consensus-spec-recommended broadcast time, leaving ~25% for gossip propagation and aggregation by the next slot's proposer) and `PAYLOAD_DUE_BPS = 75%` (validator-side observation cutoff after which envelopes do not flip `payload_present` to `True` per the Gloas validator spec). `PayloadAttestationData` values (`payload_present`, `blob_data_available`, `beacon_block_root`) evolve throughout the slot as envelopes and blobs are observed, so runners should target fetch and QBFT start near 75%, as late as the DVT round budget permits, to maximize the chance `payload_present` reflects the envelope actually arriving. The consensus specs do not prescribe a start time. QBFT and broadcast may complete after 75% and still propagate; within-slot overruns reach fork-choice via the wire path but risk missing block inclusion. Past slot end the message is dropped (gossip IGNORE and fork-choice wire REJECT when `data.slot != current_slot`), and each missed vote chips at the `PTC_SIZE/2` threshold that governs whether fork-choice extends the payload.

There is no pre-consensus phase. Operator QBFT instances agree on a stripped `PayloadAttestationVote`:

```go
type PayloadAttestationVote struct {
    BeaconBlockRoot   phase0.Root `ssz-size:"32"`
    PayloadPresent    bool
    BlobDataAvailable bool
}
```

Slot is omitted because it is already pinned by the QBFT instance (same pattern as `BeaconVote`); only the observation-dependent fields need consensus. One QBFT round covers all of the cluster's local PTC-assigned validators for the slot (committee-scoped, same as `CommitteeRunner` and `AggregatorCommitteeRunner`), rather than one QBFT per validator. At signing time, each operator reconstructs the full `PayloadAttestationData` (slot injected from the duty) and produces one partial signature per local PTC validator under `DOMAIN_PTC_ATTESTER` (domain epoch = `compute_epoch_at_slot(duty.slot)`), because each `PayloadAttestationMessage` on the wire ships a validator-specific signature verified against that validator's pubkey. All partial signatures broadcast together in a single `PartialSignatureMessages` container. After reconstruction, one `PayloadAttestationMessage` per validator is submitted to the beacon node.

The value check should reject zero `BeaconBlockRoot` (a null root cannot refer to a real block). `PayloadPresent` and `BlobDataAvailable` are observation-dependent booleans and are not compared against the local BN view (see Security Considerations); `BeaconBlockRoot` is likewise not checked against the BN's head for the slot, matching existing `BeaconVote.BlockRoot` handling. PTC attestations are not in the beacon chain slashing predicate, so no slashability call is required.

This SIP adds a new beacon role `BNRolePTCAttester` and a matching runner role `RolePTCCommittee`.

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
    RolePTCCommittee RunnerRole = 7
)
```

`MapDutyToRunnerRole()` must map `BNRolePTCAttester` to `RolePTCCommittee`. A new `PTCCommitteeDuty` is introduced, reusing the existing `ValidatorDuty`:

```go
type PTCCommitteeDuty struct {
    Slot            spec.Slot
    ValidatorDuties []*ValidatorDuty
}
```

`PTCCommitteeDuty` bundles all PTC-selected validators for a given slot under one duty, so one QBFT round on the shared `PayloadAttestationVote` covers all of them rather than running a separate consensus per validator.

### 4. Modified Proposer Duty

Relevant consensus-spec references:

- [Validator block and sidecar proposal flow](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#block-and-sidecar-proposal)

Under Gloas, `produceBlockV4` replaces the pre-Gloas proposer flow; blinded blocks are removed. The beacon node returns `Gloas.BeaconBlock` on the stateful path (and on any external-build response) or `Gloas.BlockContents` on the stateless self-build path ([beacon-APIs PR #580](https://github.com/ethereum/beacon-APIs/pull/580)).

`ProposerConsensusData` is preserved: its struct shape (`Duty`, `Version`, `DataSSZ []byte`) is unchanged. `DataSSZ` carries the SSZ-encoded `Gloas.BeaconBlock`. For the stateless `BlockContents` variant, the inline envelope, blobs, and KZG proofs returned by the BN are not put through QBFT.

Although the struct shape is unchanged, [`ProposerConsensusData.GetBlockData()`](https://github.com/ssvlabs/ssv-spec/blob/main/types/consensus_data.go#L191-L237)'s per-version switch (Capella → Fulu today) needs a new `DataVersionGloas` arm that unmarshals `DataSSZ` as `Gloas.BeaconBlock`.

Pre-consensus RANDAO flow is unchanged. Post-consensus is unchanged: each operator's `PostConsensusPartialSig` packet carries one `PartialSignatureMessage` over the block root under `DOMAIN_BEACON_PROPOSER`. Publish the signed block via the existing beacon API.

**Envelope signing out of scope.** Under Gloas, the validator signs `SignedExecutionPayloadEnvelope` only in the self-build path (`bid.builder_index == BUILDER_INDEX_SELF_BUILD`, per [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)); in the external-build path the builder signs and publishes its own envelope. This SIP does not specify distributed signing of `SignedExecutionPayloadEnvelope`, on the following grounds:

- Self-build is a fallback path in ePBS, rare in practice. Mainnet validator behavior continues to migrate away from local block building.
- Gloas introduces a trustless `execution_payload_bid` p2p market that gives proposers a new fallback for trusted block building without depending on their own EL.
- Adding a second QBFT duty solely to cover self-build envelope signing is significant protocol surface for a use case unlikely to materialize at scale among SSV operators.

Consequence: SSV proposer slots that resolve to self-build (no acceptable external bid arrived at the BN) will see PTC attestations record `payload_present = FALSE` (see §3) and the proposer forfeits the payload reward for that slot.

### 5. Proposer Preferences Duty

Relevant consensus-spec references:

- [Broadcasting SignedProposerPreferences](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/validator.md#broadcasting-signedproposerpreferences)
- [`SignedProposerPreferences` container](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/p2p-interface.md#new-proposerpreferences)
- [`proposer_preferences` gossip topic](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/p2p-interface.md#proposer_preferences)
- [`execution_payload_bid` gossip validation](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/p2p-interface.md#execution_payload_bid)

Under Gloas, each proposer broadcasts `SignedProposerPreferences` on the `proposer_preferences` p2p topic for future proposal slots within the proposer lookahead (the current epoch up to `MIN_SEED_LOOKAHEAD` epochs ahead). The signed `ProposerPreferences` carries `dependent_root`, `proposal_slot`, `validator_index`, `fee_recipient`, and `target_gas_limit`. `dependent_root` pins the proposer-lookahead epoch's seed via `get_proposer_dependent_root(state, epoch)`; operators populate it from the `dependent_root` returned by [`GET /eth/v2/validator/duties/proposer/{epoch}`](https://github.com/ethereum/beacon-APIs/pull/563) for the proposal-slot's epoch. Builders listen to this topic and use a proposer's preferences to construct `execution_payload_bid` objects for that proposer's slots. This replaces the pre-Gloas out-of-band relay-registration mechanism, which is gone along with blinded blocks.

Gossip enforces the handshake at the `execution_payload_bid` topic: each bid requires a matching `SignedProposerPreferences` for its `(proposal_slot, dependent_root)` (otherwise IGNORE'd, not forwarded). The bid `fee_recipient` must match the preference (mismatch is REJECT'd), and the bid `gas_limit` must be EIP-1559-compatible with the proposer's `target_gas_limit` via `is_gas_limit_target_compatible` (incompatible is IGNORE'd). Without this duty broadcast, bids for the validator's slots don't propagate across the network, leaving the BN with no trustless external builder options to return.

The flow matches the existing `ValidatorRegistration` / `VoluntaryExit` shape: validator-scoped, non-QBFT, one round of partial signatures, reconstruct, submit. Each operator signs `ProposerPreferences` under `DOMAIN_PROPOSER_PREFERENCES` with the validator's BLS share key. `fee_recipient` lives on `Share` (cluster-consistent); `target_gas_limit` lives in operator config (`DefaultGasLimit = 30_000_000` default with runtime overrides); `dependent_root` is observed per-operator from the BN's v2 proposer-duties endpoint. Any of the three diverging across operators would fail reconstruction (same as `ValidatorRegistration` today). `fee_recipient` is cluster-consistent in practice; the practical divergence risks are `target_gas_limit` (operator config) and `dependent_root` (observation timing around reorgs and epoch boundaries).

Trigger: at each epoch boundary, and on duty-dependent-root changes for any epoch in the proposer lookahead, iterate local validators and emit one duty per slot returned by `get_upcoming_proposal_slots(state, validator_index)`. In the `MIN_SEED_LOOKAHEAD` epochs immediately before `GLOAS_FORK_EPOCH`, this SIP requires operators to emit preferences for any local-validator proposal slots in the first Gloas epoch. The semantics of `get_upcoming_proposal_slots` plus the gossip rule that `preferences.proposal_slot` must be within the proposer lookahead leave no other emission window for those slots; this aligns with the spec's *"Proposers SHOULD broadcast their preferences in the epoch before the fork"* recommendation in `p2p-interface.md`. The `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple; emission-timing implications are covered in Security Considerations. If the proposer lookahead for an epoch changes, cached duties for that epoch are replaced rather than merged.

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
    ProposerPreferencesPartialSig PartialSigMsgType = 7
)
```

`MapDutyToRunnerRole()` must map `BNRoleProposerPreferences` to `RoleProposerPreferences`.

## Security Considerations

### `GloasBeaconVoteValueCheckF` must include `AttestationDataIndex` in slashability checks

Under Gloas, `AttestationData.Index` is part of the attestation data root and therefore part of the double-vote slashing predicate. `GloasBeaconVoteValueCheckF` must reconstruct the full Gloas `AttestationData` with `Index` from the decided `GloasBeaconVote.AttestationDataIndex` before calling `IsAttestationSlashable`; otherwise an operator could sign `index=0` and `index=1` for the same `(source, target)` in the same slot without the predicate tripping.

### Payload-status fields are trusted from the QBFT leader

Value checks for Gloas `AttestationData.Index` and PTC `payload_present` / `blob_data_available` do not require the decided value to match each operator's local BN view. Requiring local agreement would fail QBFT rounds whenever operators observe the envelope at slightly different times around the 75% deadline, a normal gossip-lag scenario. Accepted tradeoff: a malicious QBFT leader can push a value contrary to the cluster's majority BN observation. This matches existing ssv-spec treatment of `BeaconVote.BlockRoot`, which is trusted from the leader because BNs legitimately diverge on fork-choice head.

### Config divergence silently disables trustless external builder bids

`ProposerPreferences` reconstruction requires cluster-wide agreement on `target_gas_limit` (per-operator config) and `dependent_root` (per-operator BN observation). Divergence on either produces no reconstructed signature, no gossip publication, and therefore no matching preference on the `execution_payload_bid` topic; bids for the slot are IGNORE'd by gossip (§5), leaving the BN with no trustless external builder options to return. Same reconstruction failure shape as `ValidatorRegistration` today.

### Too-early `SignedProposerPreferences` publication pins the wrong preference

Because the `proposer_preferences` gossip topic accepts only the first valid message per `(dependent_root, proposal_slot, validator_index)` tuple (§5), reconstructing and publishing a preference before all tuple inputs are final can durably pin a wrong-input preference: a later corrected message for the same tuple is dropped by gossip rather than treated as a replacement. Builders keep using the stale preference, and bids matching the corrected values fail the §5 handshake. Operators must therefore hold publication until `dependent_root`, `fee_recipient`, and `target_gas_limit` are all final for the tuple, and re-emit only when the tuple itself changes (notably when `dependent_root` shifts due to reorg, or `proposer_lookahead` reassigns the validator to a different slot). Distinct from the config-divergence entry above: there, divergence prevents publication; here, premature publication pins the wrong preference more durably than no publication would.

### Self-build slots produce `payload_present = FALSE`

Because this SIP omits distributed envelope signing (§4), slots where the proposer's BN falls back to self-build will see PTC attestations record `payload_present = FALSE` (§3) and the proposer will forfeit the payload reward boost for that slot. If self-build prevalence becomes a material liveness concern post-Gloas mainnet activation, distributed envelope signing can be added in a follow-up SIP without breaking this baseline.

## Open Questions / Upstream Watchlist

This section is intentionally limited to upstream items that could still change the normative SSV behavior described above. If any of these settle differently, this SIP should be updated.

- PTC Beacon API detail drift: the core PTC duty, payload-attestation-data, and pool-submission surfaces are already present in upstream Beacon API `master`, and this SIP assumes those mainline shapes. Watch them for any remaining field, header, or duty-refresh semantic changes.
- `produceBlockV4` shape stabilization: this SIP relies on the current reviewed shape of `produceBlockV4` (`apis/validator/block.v4.yaml`), which has not been merged to `beacon-APIs/master` yet — it lives in [PR #580](https://github.com/ethereum/beacon-APIs/pull/580). Watch for changes to the response variant discriminator (stateful `BeaconBlock` vs stateless `BlockContents`) and the block submission wrapper shape. The PR's discussion also covers restructuring `builder_boost_factor` for multi-builder connections; this will reshape SSV node-operator config but does not change SIP-normative behavior.
- `head_v2` SSE event ([PR #590](https://github.com/ethereum/beacon-APIs/pull/590), Gloas-labeled upstream): adds an ePBS-specific `payload_status` field (`empty` / `full`) that may update mid-slot as the envelope is observed, and renames `{previous,current}_duty_dependent_root` → `{previous,current}_epoch_dependent_root` while adding `next_epoch_dependent_root`. Both §3 (PTC duty refresh) and §5 (proposer-preferences re-emission trigger) key off these dependent-root fields, so implementations will need to consume the renamed fields once the PR lands. `payload_status` is also optionally useful as an early signal for the PTC runner's internal decision cutoff (§3).
- Validator-facing `SignedProposerPreferences` publication endpoint: the Gloas validator spec expects validators to broadcast preferences to the [`proposer_preferences`](https://github.com/ethereum/consensus-specs/blob/4a4937bea332d72a55a76aaebcb97fbcdc189f69/specs/gloas/p2p-interface.md#proposer_preferences) gossipsub topic, but `beacon-APIs/master` does not yet expose a validator-facing publication endpoint. §5 is specified against a future `SubmitProposerPreferences(...)` BN abstraction method whose concrete Beacon API shape is TBD.
