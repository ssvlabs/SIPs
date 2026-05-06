|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Shane Moore    | ePBS (EIP-7732) Support    | Core       | draft               | 2026-04-18 |

## Summary

Describes the SSV spec changes needed to keep SSV operators performing validator duties correctly after epbs, [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732), is implemented in Ethereum's consensus layer Gloas fork. Based on the pinned [Gloas consensus-spec snapshot](https://github.com/ethereum/consensus-specs/tree/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas) (`ethereum/consensus-specs@f1371480c4`, reviewed 2026-04-17).

Validator client related changes via ePBS:
1. earlier slot deadlines
2. attestation handling changes, including preservation of the Gloas `attestation.data.index` semantics
3. the new Payload Timeliness Committee (PTC)
4. the Gloas proposer flow using `produceBlockV4`, which returns either `Gloas.BlockContents` (self-build: block + inline execution-payload envelope + blobs + KZG proofs) or `Gloas.BeaconBlock` (external-builder: block only, envelope published by the builder)
5. the new `SignedProposerPreferences` message must be submitted if the node operator wants to be able to select block bids received over p2p

## Motivation

Gloas changes validator duties in ways that break a few current SSV assumptions:

- attestation `index` is no longer safely reconstructible from local validator duty data
- proposer post-consensus can require signing more than one beacon object for the same duty
- the new PTC duty has a late in-slot deadline
- SSV validators must broadcast `SignedProposerPreferences` or they cannot accept external-builder bids for their slots

## Rationale

Key design choices and why:

- **`BeaconVote` gains `AttestationDataIndex`.** In Gloas, `AttestationData.Index` is BN-supplied and part of the signed attestation root, so it must travel through QBFT consensus data rather than being reconstructed locally.
- **PTC is a committee-scoped runner.** `PayloadAttestationData` is validator-independent (like `BeaconVote`), while each PTC-assigned validator still needs its own BLS signature and submission object. This matches the existing committee-runner pattern from `committee_consensus.md`.
- **Proposer-preferences is validator-scoped and non-QBFT.** `fee_recipient` already lives per-validator on `Share.FeeRecipientAddress`; `gas_limit` lives in operator config (currently `DefaultGasLimit = 30_000_000` in `types/beacon_types.go`, with runtime overrides, same as the existing validator-registration flow). The signed object is therefore agreed off-chain, so there is nothing to reach consensus over. The registration-like one-round partial-sig-and-submit flow from `voluntary_exit.md` fits directly.
- **`ProposerConsensusData` carries raw SSZ bytes rather than split fields.** QBFT agrees on the exact `produceBlockV4` response, keeping SSV aligned with the Beacon API wire output and side-stepping ad-hoc field maps for two variants (`BeaconBlock` vs `BlockContents`).

## Specification

### 1. Slot Timing Changes

All existing validator duty deadlines shift earlier in the slot. A new PTC deadline is added.

Relevant consensus-spec references:

- [Validator time parameters](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/validator.md#time-parameters)

| Duty | Pre-ePBS | Post-ePBS (Gloas) |
|------|----------|--------------------|
| Attestation | 1/3 slot (~4s) | 1/4 slot (25%, ~3s) |
| Sync Committee Message | 1/3 slot | 1/4 slot (25%) |
| Aggregation | 2/3 slot (~8s) | 1/2 slot (50%, ~6s) |
| Sync Committee Contribution | 2/3 slot | 1/2 slot (50%) |
| PTC Attestation | - | 3/4 slot (75%, ~9s) |

### 2. Modified Attestation Duty

Relevant consensus-spec references:

- [Validator attestation changes](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/validator.md#attestation)

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

`BeaconVote` gains an `AttestationDataIndex` field for Gloas, matching the `phase0.CommitteeIndex` (a `uint64` alias) type of `AttestationData.Index` in consensus specs so reconstruction is a direct field assignment. The restricted Gloas value space (`0` = `EMPTY`, `1` = `FULL` for non-same-slot attestations; `0` for same-slot) is enforced in the value check below, not at the type level.

```go
// Current (ssv-spec types/consensus_data.go)
type BeaconVote struct {
    BlockRoot phase0.Root `ssz-size:"32"`
    Source    *phase0.Checkpoint
    Target    *phase0.Checkpoint
}

// Gloas-extended
type BeaconVote struct {
    BlockRoot            phase0.Root        `ssz-size:"32"`
    Source               *phase0.Checkpoint
    Target               *phase0.Checkpoint
    AttestationDataIndex phase0.CommitteeIndex // copied from AttestationData.Index
}
```

#### Value check

`BeaconVoteValueCheckF()` becomes fork-aware:

- pre-Gloas: reject non-canonical `AttestationDataIndex` values; existing slashability workaround otherwise unchanged.
- Gloas and later: reject `AttestationDataIndex` values other than `0` or `1`; build slashability checks using the real `AttestationDataIndex`.

#### Implementation note: aggregation path

The `BNRoleAggregator` duty (handled by the aggregator-committee runner) fetches aggregated attestations from the Beacon API's aggregate-attestation endpoint with `attestation_data_root` as an input. Implementations must compute that root from the BN-supplied `AttestationData` (including its Gloas `index`).

### 3. New Duty: Payload Timeliness Committee (PTC) Attestation

Relevant consensus-spec references:

- [Validator payload timeliness attestation flow](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/validator.md#payload-timeliness-attestation)
- [Beacon-chain payload attestation containers](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/beacon-chain.md#payloadattestationdata)
- [Fork-choice payload attestation deadline](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/fork-choice.md#new-get_payload_attestation_due_ms)

PTC is a per-slot consensus-layer-selected set of validators that attests to payload and blob availability for the slot's beacon block before the deadline.

Each validator signs a `PayloadAttestationData` object carrying `beacon_block_root`, `slot`, `payload_present`, and `blob_data_available`, then submits a validator-specific `PayloadAttestationMessage(validator_index, data, signature)` to the beacon node.

At the start of each epoch, SSV should fetch PTC duties for the next epoch and refresh them on duty-dependent-root changes. Because PTC duty responses may be sparse and incomplete, a changed duty-dependent root for an epoch should replace the cached duties for that epoch rather than being merged.

`PAYLOAD_ATTESTATION_DUE_BPS = 75%` is the consensus-spec-recommended broadcast time, chosen to leave ~25% of the slot for gossip propagation and aggregation by the next slot's proposer. `PayloadAttestationData` values (`payload_present`, `blob_data_available`, `beacon_block_root`) evolve throughout the slot as envelopes and blobs are observed, so runners should delay fetch and QBFT start as late as the DVT round budget permits to maximize the chance `payload_present` reflects the envelope actually arriving. The consensus specs do not prescribe a start time. Within-slot overruns past 75% still reach fork-choice via the wire path but risk missing block inclusion; past slot end the message is dropped (gossip IGNORE, fork-choice wire REJECT on `data.slot == current_slot`), and each missed vote chips at the `PTC_SIZE/2` threshold that governs whether fork-choice extends the payload.

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

- [Validator block and sidecar proposal flow](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/validator.md#block-and-sidecar-proposal)
- [Builder execution payload envelope construction](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/builder.md#constructing-the-signedexecutionpayloadenvelope)

Under Gloas, `produceBlockV4` replaces the pre-Gloas proposer flow; blinded blocks are removed. The beacon node now returns one of two variants:

- `Gloas.BlockContents` (self-build): beacon block plus inline envelope, blobs, and KZG proofs. The validator signs both the block and the `ExecutionPayloadEnvelope`.
- `Gloas.BeaconBlock` (external-builder bid accepted by the beacon node): block only. The builder later signs and publishes its own `SignedExecutionPayloadEnvelope`.

`ProposerConsensusData` is preserved: its struct shape (`Duty`, `Version`, `DataSSZ []byte`) is unchanged, because it already carries raw SSZ bytes. The Gloas work lives entirely inside `GetBlockData()`, which grows a `DataVersionGloas` case that decodes into either `Gloas.BlockContents` (self-build) or `Gloas.BeaconBlock` (external-builder), exposing the block object and, when the decoded variant is `BlockContents`, the inline envelope plus blobs and KZG proofs.

Post-consensus object rules:

- both objects (block + envelope) are signed by the validator's existing BLS share key; the envelope changes the domain, not the signer identity
- each operator's `PostConsensusPartialSig` packet must carry one `PartialSignatureMessage` per required root for the decided variant:
  - `Gloas.BeaconBlock`: one entry (block root)
  - `Gloas.BlockContents`: two entries (block root + envelope root)

Pre-consensus RANDAO flow is unchanged. Operators run QBFT on `ProposerConsensusData`, then sign the objects required by the decided variant.

Publication order and completion:
- publish the signed beacon block first
- if the decided value is `Gloas.BlockContents` and block publication succeeded, publish the envelope wrapper immediately after, before the PTC 75% deadline so it can influence timely payload attestations

### 5. Proposer Preferences Duty

Relevant consensus-spec references:

- [Broadcasting SignedProposerPreferences](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/validator.md#broadcasting-signedproposerpreferences)
- [`SignedProposerPreferences` container](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/p2p-interface.md#new-proposerpreferences)
- [`proposer_preferences` gossip topic](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/p2p-interface.md#proposer_preferences)
- [`execution_payload_bid` gossip validation](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/p2p-interface.md#execution_payload_bid)

Under Gloas, each proposer declares their preferred `fee_recipient` and `gas_limit` for upcoming proposal slots (future slots in the current epoch plus all proposal slots in the next epoch) by broadcasting `SignedProposerPreferences` on the `proposer_preferences` p2p topic. Builders listen to this topic and use a proposer's preferences to construct `execution_payload_bid` objects for that proposer's slots. This replaces the pre-Gloas out-of-band relay-registration mechanism, which is gone along with blinded blocks.

Gossip enforces the handshake at the `execution_payload_bid` topic: bids for a slot with no seen `SignedProposerPreferences` are IGNORE'd (not forwarded), and bids whose `fee_recipient` or `gas_limit` disagree with the proposer's preferences are REJECT'd. Without this duty broadcast, bids for the validator's slots don't propagate across the network, leaving the BN with no trustless external builder options to return.

The flow matches the existing `ValidatorRegistration` / `VoluntaryExit` shape: validator-scoped, non-QBFT, one round of partial signatures, reconstruct, submit. Each operator signs `ProposerPreferences` under `DOMAIN_PROPOSER_PREFERENCES` with the validator's BLS share key. `fee_recipient` lives on `Share` (cluster-consistent); `gas_limit` lives in operator config (`DefaultGasLimit = 30_000_000` default with runtime overrides), so operators must configure matching values out-of-band. Divergence on `gas_limit` fails reconstruction, same as `ValidatorRegistration` today.

Trigger: at each epoch boundary, and on duty-dependent-root changes for the current or next epoch, iterate local validators and emit one duty per slot returned by `get_upcoming_proposal_slots(state, validator_index)` (future slots in the current epoch plus all slots of the next epoch). In the epoch immediately before `GLOAS_FORK_EPOCH`, operators must also emit preferences for any local-validator proposal slots in the first Gloas epoch; otherwise bids for those slots will not propagate during the first post-fork epoch (per the pre-fork subscription note in `p2p-interface.md`). Per the Gloas validator spec, a validator MAY broadcast multiple `SignedProposerPreferences` for the same slot (later messages supersede earlier ones), so this SIP does not hard-cap the number of `SignedProposerPreferences` publications per slot. If the proposer lookahead for an epoch changes, cached duties for that epoch are replaced rather than merged.

This SIP adds a new beacon role `BNRoleProposerPreferences`, a matching runner role `RoleProposerPreferences`, and a new `PartialSigMsgType` `ProposerPreferencesPartialSig`.

```go
// types/beacon_types.go additions
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

### `BeaconVoteValueCheckF` must include `AttestationDataIndex` in slashability checks

Under Gloas, `AttestationData.Index` is part of the attestation data root and therefore part of the double-vote slashing predicate. `BeaconVoteValueCheckF` must reconstruct the full Gloas `AttestationData` with `Index` from the decided `BeaconVote.AttestationDataIndex` before calling `IsAttestationSlashable`; otherwise an operator could sign `index=0` and `index=1` for the same `(source, target)` in the same slot without the predicate tripping.

### Payload-status fields are trusted from the QBFT leader

Value checks for Gloas `AttestationData.Index` and PTC `payload_present` / `blob_data_available` do not require the decided value to match each operator's local BN view. Requiring local agreement would fail QBFT rounds whenever operators observe the envelope at slightly different times around the 75% deadline, a normal gossip-lag scenario. Accepted tradeoff: a malicious QBFT leader can push a value contrary to the cluster's majority BN observation. This matches existing ssv-spec treatment of `BeaconVote.BlockRoot`, which is trusted from the leader because BNs legitimately diverge on fork-choice head.

### Config divergence silently disables trustless external builder bids

`ProposerPreferences` reconstruction requires cluster-wide agreement on `gas_limit`, which lives in per-operator config rather than `Share`. Divergence produces no reconstructed signature, no gossip publication, and therefore no trustless external builder bids for that slot (the `execution_payload_bid` topic IGNOREs bids with no matching preferences). Same reconstruction failure shape as `ValidatorRegistration` today.

## Open Questions / Upstream Watchlist

This section is intentionally limited to upstream items that could still change the normative SSV behavior described above. If any of these settle differently, this SIP should be updated.

- PTC Beacon API detail drift: the core PTC duty, payload-attestation-data, and pool-submission surfaces are already present in upstream Beacon API `master`, and this SIP assumes those mainline shapes. Watch them for any remaining field, header, or duty-refresh semantic changes.
- Self-build proposer API stabilization: this SIP relies on the current reviewed shape of `produceBlockV4` (`apis/validator/block.v4.yaml`) and stateless envelope publication (`apis/beacon/execution_payload/envelope_post.yaml`), neither of which has been merged to `beacon-APIs/master` yet; both live in [PR #580](https://github.com/ethereum/beacon-APIs/pull/580). Watch for changes to self-build versus block-only response behavior, required submission wrappers, or publication headers. The PR's discussion also covers restructuring `builder_boost_factor` for multi-builder connections; this will reshape SSV node-operator config but does not change SIP-normative behavior (the §4 decode / value-check / post-consensus rules remain variant-agnostic).
- `head_v2` SSE event ([PR #590](https://github.com/ethereum/beacon-APIs/pull/590), Gloas-labeled upstream): adds an ePBS-specific `payload_status` field (`empty` / `full`) that may update mid-slot as the envelope is observed, and renames `{previous,current}_duty_dependent_root` → `{previous,current}_epoch_dependent_root` while adding `next_epoch_dependent_root`. Both §3 (PTC duty refresh) and §5 (proposer-preferences re-emission trigger) key off these dependent-root fields, so implementations will need to consume the renamed fields once the PR lands. `payload_status` is also optionally useful as an early signal for the PTC runner's internal decision cutoff (§3).
- Validator-facing `SignedProposerPreferences` publication endpoint: the Gloas validator spec expects validators to broadcast preferences to the [`proposer_preferences`](https://github.com/ethereum/consensus-specs/blob/f1371480c4da884398e688d81b030f5280a6a578/specs/gloas/p2p-interface.md#proposer_preferences) gossipsub topic, but `beacon-APIs/master` does not yet expose a validator-facing publication endpoint. §5 is specified against a future `SubmitProposerPreferences(...)` BN abstraction method whose concrete Beacon API shape is TBD.
