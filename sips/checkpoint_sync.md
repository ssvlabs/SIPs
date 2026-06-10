| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Diego Marin Santos ([@diegomrsantos](https://github.com/diegomrsantos)) | Canonical State Root and Checkpoint Sync | Core | open-for-discussion | (none) | 2026-06-09 |

[Discussion](https://github.com/ssvlabs/SIPs/discussions/93)

**Summary**  

Every SSV node keeps a copy of the network state: the list of operators, validators, and clusters. A node builds that state by folding the SSV smart contract's events in order, starting at the deployment block. The fold needs only the event log and no contract storage reads at old blocks.

On mainnet, however, that means obtaining and replaying the full log history back to the 2023 deployment block. A checkpoint lets a node start from a recent finalized block without doing that replay, while reconstruction remains available as the audit fallback for anyone who can still obtain the logs.

The design is the Candidate A path from the discussion: a signed checkpoint that changes no contract. Candidate B would instead publish the state root on the SSV contract itself. It is stronger but needs a contract change and is not specified here.

A trusted checkpoint has one hard problem. A payload can parse cleanly and still leave out records or describe them wrongly, and the node has no way to notice the missing records unless it checks a commitment to the complete state. This SIP calls that the completeness problem. Spot checks with eth_call can only validate records the node already knows about; they cannot prove that nothing was left out.

To catch missing public records this SIP uses a canonical state root. A state root is one hash that stands for the entire network state at a block. Canonical means every correct client computes the exact same hash from the same logical input. If two clients, built by two different teams, compute the same root from the same correct event scope and canonical spec, then one implementation cannot silently drop a record without changing the hash. Matching roots prove agreement on the committed record set; they do not prove the event scope or canonical spec is correct, so the full fold audit remains the backstop for shared mistakes. Independent signers compute `state_root` and `share_set_root` from retained state, from a historical materializer, or from a parent checkpoint plus new logs, and sign a certificate message containing those roots.

The core protocol surface has three parts: (1) the canonical network state and how to encode it, (2) the checkpoint payload format for public state and encrypted shares, and (3) the certificate format and what a certificate is required to mean. The SIP also specifies the client import and verification flow and a conformance test that guards the canonical spec.

Throughout this SIP, the term logs means the SSV smart contract's event log, which any full node serves over eth_getLogs.

**Rationale & Design Goals**  

Two independent implementations that fold the same logs should produce the same bytes. Today they do not always agree. Anchor issue [#972](https://github.com/sigp/anchor/issues/972) is a live example: Anchor and go-ssv decode an OperatorAdded operator public key differently, which causes Anchor to silently skip an operator and diverge from the other client. Bugs of this kind are invisible until two implementations are compared byte for byte at the same block.

The design goals follow from that observation.

- A node that skips the fold must still be protected against an incomplete or misrepresented state. The protection is agreement across implementations on a canonical root, not trust in whoever published the file.
- The canonical state must be specified down to encoding, ordering, and application semantics, so that agreement is meaningful and so that the encoding choices where two clients can silently disagree (such as the one behind #972) are pinned down in one place rather than rediscovered by each client team.
- Checkpoint sync removes the need to replay all history at startup while preserving reconstruction from the deployment block as the audit path for anyone who can still obtain the logs.
- The format must allow the signature scheme to be replaced later without a format break.
- The mechanism must commit at blocks that are permanent, so that no reorg can invalidate a published checkpoint.

The sequencing matters. The canonical state spec is the precondition for everything else. The differential conformance check enforces it. Checkpoints and certificates sit on top.

**Specification**  

This version defines a closed v1 canonical profile. A checkpoint accepted under `canonical_spec_version = 1` is valid only under the rules below. Any alternative root shape, operator identity rule, removal representation, operator public key decoding rule, or nonce transition is a new canonical spec version, not an implementation choice inside v1.

**1. Canonical Global State**

To build the canonical global state at block B, start with empty state and apply every relevant SSV event in block order, from the deployment block through block B. The result is the canonical global state at B. The relevant events are OperatorAdded, OperatorRemoved, ValidatorAdded, ValidatorRemoved, ClusterLiquidated, ClusterReactivated, and FeeRecipientAddressUpdated. ValidatorExited is deliberately not in this set; see Excluded from the state root.

The canonical state root is a pure function of this fold. It is defined over the record set included in scope in canonical form, not over any client's database or representation in memory. A conforming implementation must be able to materialize the full canonical record set before hashing, no matter what it keeps at runtime. Storage choices are explicitly outside the hashed state: whether a client retains only its own shares, whether it deletes a removed record or keeps a flagged tombstone, and similar optimizations must not change the root.

The block number is a reference point that records when the state was taken. It is not part of the hashed state itself.

**Records in scope.** The following records are derived from the logs and are identical for all conforming nodes.

| Record | Fields |
| ------ | ------ |
| Operator | operator_id (sequential, assigned by the contract), canonical_public_key, owner address, removed status |
| Validator | validator public key (BLS), cluster it belongs to |
| Cluster | cluster_id (derived, see below), owner address, sorted member operator_ids, liquidated flag |
| Owner | owner address, fee_recipient, next_validator_nonce |

**Counters are not a substitute for records.** The canonical root must commit to the full set of records included in scope by content. It must not substitute any monotonic counter, count, or high water mark for the record set. A counter such as a highest operator id seen value can advance on both clients while one of them is missing an operator's record, which defeats completeness. #972 is exactly this trap: the omitted operator changes the operator record set, and the root catches it only because the root commits to records, not to the operator id counter.

**Application semantics are part of the canonical state.** The canonical state is defined by the encoding and ordering below, plus the application semantics: which events are accepted versus ignored, the signature and nonce validation rules, and the effect of a rejected event. A root mismatch on identical logs can therefore come from a decoding bug or from a difference in application semantics, and both are in scope for this spec. V1 pins those rules in this section; a client must not substitute its local storage behavior for these canonical fold rules.

**Owner state and validator nonces.** The canonical Owner record stores `next_validator_nonce`, the nonce value to use for the next ValidatorAdded by that owner. It does not store a client's local representation of the last consumed nonce, and it does not depend on whether a client stores that value as nullable. This distinguishes an owner that only changed its fee recipient, whose next validator nonce is still 0, from an owner that already consumed nonce 0.

An absent Owner record means the effective defaults: `fee_recipient = owner` and `next_validator_nonce = 0`. A FeeRecipientAddressUpdated event changes the effective `fee_recipient` and leaves `next_validator_nonce` unchanged, using 0 if no previous owner nonce state exists. A ValidatorAdded event is evaluated against the current `next_validator_nonce` for that owner and then advances `next_validator_nonce` by one before the rest of the event is validated. If the ValidatorAdded event is rejected after this point, the nonce advance remains and no validator record, cluster update, or encrypted share records are created. An Owner record is included in the canonical record set when either `fee_recipient != owner` or `next_validator_nonce != 0`; otherwise it is omitted because it is identical to the default state.

**Excluded from the state root.** The validator index on the beacon chain is fetched from the beacon node, not derived from the logs, so it is not a deterministic function of the logs and is not part of the v1 state root. A future canonical spec version may add a field derived from beacon data, but v1 excludes it.

ValidatorExited is also excluded. It is a beacon chain signal: it announces that a validator is exiting the beacon chain, not that its SSV registry record has changed. A validator's registry record is removed only by ValidatorRemoved; ValidatorExited leaves the operator, validator, and cluster records untouched. It therefore changes no canonical record and no root. This is distinct from the operator removed status above, which is set only by OperatorRemoved.

The operator fee and the Cluster accounting fields carried by each event (balance, validatorCount, networkFeeIndex, index, and the active flag) are also excluded: they are mutable contract bookkeeping that varies over time, not part of the canonical state for validator clients, and committing to accumulators that advance on nearly every interaction would change the root far more often than the logical membership state does. Cluster liquidation status is derived from the ClusterLiquidated and ClusterReactivated events, not from the Cluster.active field. In v1, those events update only an included cluster record. If no cluster record is included for that owner and operator set, the event changes no canonical record. A cluster created later starts with `liquidated = false` unless a later ClusterLiquidated event changes it.

**Removal representation.** V1 pins removal per entity. A ValidatorRemoved event removes the validator record from `state_root` and removes the validator's encrypted share records from `share_set_root`. If that was the last active validator in a cluster, the cluster record is also omitted. An OperatorRemoved event sets `removed = true` for that operator while any active cluster still references it. A removed operator with no active cluster reference is omitted from the canonical record set. Active operators are included even if they are not members of an active cluster.

**Canonical encoding and ordering.** The following choices are pinned.

- Operator identity. Operators are keyed by `operator_id` only. The canonical fold must not deduplicate operators by raw event bytes, decoded RSA key bytes, PEM text, owner address, or any other key. If two OperatorAdded events assign two different operator ids but decode to the same RSA public key, v1 still has two operator records.
- Operator public key. OperatorAdded publicKey bytes have appeared in more than one layout, so v1 defines `canonical_public_key` explicitly. First, try to decode the field as the SSV operator public key wrapper, equivalent to ABI decoding one dynamic `bytes` value. If that succeeds and the decoded bytes parse as the expected base64 PEM RSA public key payload, those decoded bytes are the canonical bytes. If wrapper decoding fails, the raw event bytes are accepted only if they parse directly as the same base64 PEM RSA public key payload. Otherwise the OperatorAdded event is rejected as malformed. The canonical bytes are hashed exactly as bytes; clients must not reserialize the key into a different PEM, DER, JSON, or text layout before hashing. If a client cannot decode the key into this canonical form, it must stop or mark the event malformed according to the v1 fold, not silently skip the operator.
- Cluster member set. Operator ids in a cluster are an unordered set in the contract. They are serialized in ascending operator_id order.
- Addresses. All addresses are encoded as their 20 raw bytes when hashed. Any text rendering uses lowercase hex. EIP-55 checksum casing is not used in the hashed form.
- Fee recipient fallback. If no Owner record exists for an owner, the fee recipient defaults to the owner address.
- Owner nonce. Owner records serialize `next_validator_nonce`, the nonce value to use for the next ValidatorAdded by that owner. It starts at 0. Every ValidatorAdded log for the owner advances it exactly once before validation, including a malformed or rejected ValidatorAdded.
- BLS public key. Validator public keys are encoded as their raw compressed bytes when hashed.
- Root domain. The root serialization begins with a domain header containing `STATE_ROOT_V1`, `canonical_spec_version = 1`, `network_id`, and `ssv_contract_address` from the certificate message. This prevents a state root from one network or SSV contract from being reused under another.

**Cluster id derivation.** The cluster id is derived, not stored, and must match across implementations:

```text
cluster_id = keccak256(
    owner_address_bytes (20 bytes)
    ++ for each operator_id in ascending sorted order:
        ( 24 zero bytes ++ operator_id as 8-byte big-endian )
)
```

**2. State Root Construction**

V1 `state_root` is `keccak256` over the v1 root domain header followed by the canonical serialization of all records from section 1. This is the only `state_root` construction accepted under `canonical_spec_version = 1`. keccak256 is chosen to align with Ethereum tooling and with the cluster_id derivation already defined in section 1.

Record ordering for the root is fully defined so the serialization is deterministic. These orderings at the top level are normative for v1 and must be covered by the conformance tests in section 6.

- Operators are ordered by ascending operator_id.
- Validators are ordered by their canonical BLS public key bytes.
- Clusters are ordered by their derived cluster_id bytes.
- Owners are ordered by their address bytes.
- Within a cluster, the member operator_ids are in ascending order, as in the cluster_id derivation.

Two framing rules are normative for any construction below, because keccak256 over a bare concatenation of heterogeneous records is ambiguous: an address with 20 bytes followed by an id with 8 bytes is identical at the byte level to a single field with 28 bytes. The rules:

- Length prefixes. Every field with variable length and every record is prefixed with its length so its boundaries are unambiguous. Fields with fixed width (the address with 20 bytes, the 8 byte big endian operator_id, the compressed BLS key) may be written raw because their width is fixed and documented.
- Domain separation by record kind. Each record kind (Operator, Validator, Cluster, Owner) carries a fixed kind tag in its serialization, so bytes of one kind can never be interpreted again as another kind at the same root.

These are not tree shape details; they are required even for the flat hash. A Merkle tree for inclusion proofs is a future canonical spec version because its exact shape must be specified before it can produce a protocol root.

**3. Checkpoint Format**

A checkpoint is a certificate plus one or more untrusted payloads. The certificate signs canonical commitments to public state and active encrypted shares. The payloads carry the preimages needed by an importing node to rebuild those commitments without replaying old logs.

The certificate does not sign one specific snapshot file, compression format, chunking scheme, mirror, or JSON or SSZ layout. Any file layout is acceptable if the importing node can parse it and recompute the signed canonical commitments from its contents.

**Snapshot payload.** The snapshot carries the canonical global state at block B, serialized in the canonical form of section 1, so that an importing node can populate its storage without folding the logs.

The snapshot also carries the full active encrypted share set. Each ValidatorAdded event publishes, for every operator in the cluster, a share public key and an encrypted key share. Those ciphertexts are public event data, while the plaintext share remains protected by encryption to the operator's key. A fresh operator bootstrap needs its encrypted share, and client retention differs, so this set is not left to local storage. Therefore the certificate includes a mandatory `share_set_root`, separate from `state_root`.

`state_root` commits to the public validator client registry state. `share_set_root` commits to the encrypted share records for validators active at block B. It does not commit to encrypted shares for validators that were added and removed before B, because those shares are not needed to bootstrap the current state.

The `share_set_root` uses the same framing rules as section 2, with its own root domain header and record kind tag. Its domain header contains `SHARE_SET_ROOT_V1`, `canonical_spec_version = 1`, `network_id`, and `ssv_contract_address` from the certificate message. V1 encrypted share records are ordered by validator public key bytes and then by ascending operator_id. Each record contains:

```text
ENCRYPTED_SHARE_RECORD_V1
owner
validator_public_key
cluster_id
operator_id
share_public_key
encrypted_share_bytes
```

The `cluster_id` is derived from owner and the sorted operator set as defined in section 1. The `encrypted_share_bytes` field carries a length prefix because its size is variable. This root proves share completeness and byte integrity for bootstrap; it does not require signers to decrypt shares.

**Checkpoint production paths.** The protocol is defined by the records and roots, not by a particular client's database. A process can produce a checkpoint only if it can enumerate the canonical public state and active encrypted share records at block B. That process may be a normal client that retained those records, a client with added retention, a dedicated materializer, or an archive indexer.

For the initial checkpoint, where `parent_checkpoint_hash` is zero, there are two valid production paths:

1. Retained state path. The producer already has a complete current registry snapshot at B, including active encrypted share ciphertexts for every validator and operator pair. It exports that state and computes `state_root` and `share_set_root` directly from the canonical records.
2. Historical materializer path. The producer does not have retained encrypted shares. It folds historical registry events through B, materializes the same active public state and encrypted share set, discards encrypted shares for validators removed before B, and computes the same roots.

A client whose current state does not retain all active encrypted share ciphertexts cannot produce the encrypted share part of the initial checkpoint from that state alone. It must use a materializer, an archive source, or a previous checkpoint that already contains the share set.

For every checkpoint with a nonzero `parent_checkpoint_hash`, producers start from the parent checkpoint state and apply only the relevant SSV logs from the block after the parent through B. They update the public records and active encrypted share records, compute the new roots, and include `delta_log_set_hash` for that bounded event range.

The checkpoint bundle may include an optional `snapshot_digest` outside the signed certificate for download integrity, caching, or mirror comparison. That digest is not a trust anchor; import step 4 treats it only as a transport check.

**Certificate message.** The certificate message is the small object signers sign.

| Field | Meaning |
| ----- | ------- |
| schema_version | Version of the certificate message format |
| canonical_spec_version | Version of the canonical state definition: the included event set and the application and encoding semantics of section 1 that determine the root |
| network_id | Network and chain id the state belongs to |
| ssv_contract_address | Address of the SSV contract whose registry state is represented |
| block_number | Block B the state was taken at |
| block_hash | Hash of block B |
| parent_checkpoint_hash | Hash of the parent checkpoint certificate message, or zero for an initial checkpoint |
| state_root | Canonical state root from section 2 |
| share_set_root | Canonical root of the active encrypted share set |
| delta_log_set_hash | Hash of the exact ordered list of relevant SSV events consumed after the parent checkpoint, or zero for an initial checkpoint |
| signer_set_id | Identifier of the signer set that may certify this message |
| scheme_version | Identifier of the signature scheme used by certificates |

For a checkpoint with a parent, `delta_log_set_hash` binds the certificate message to the exact events consumed between the parent checkpoint and B, so that a later comparison or audit can confirm both parties read the same delta logs. For an initial checkpoint, `delta_log_set_hash` is zero. A producer may publish optional audit metadata that records a full historical log hash, but importers do not require it and it is not part of the certificate message.

The canonical_spec_version is distinct from schema_version: schema_version versions the certificate message format, while canonical_spec_version versions the canonical state definition that determines the root. The root is meaningful only under the definition that produced it, so a future SSV contract upgrade that adds an event that changes state or changes application semantics is a new canonical state generation that bumps canonical_spec_version and requires regenerating the conformance vectors of section 6. An importer rejects a canonical_spec_version it does not implement (import step 1) rather than comparing roots across generations.

**4. Certificates**

A certificate is a signature over the certificate message by a member of the signer set.

**Required meaning.** A certificate attests that the signer independently materialized the canonical public state and active encrypted share set at block B, without trusting the payload being signed, and computed this same `state_root` and `share_set_root` using one of the production paths in section 3. For a child checkpoint, it also attests that the signer consumed the exact ordered event delta identified by `delta_log_set_hash`. A certificate is not an attestation that a downloaded checkpoint file parsed or that a particular payload encoding or hosting path is trustworthy.

**Acceptance rule.** A checkpoint is accepted only when at least X of the Y signers in the signer set produce matching certificates over the same certificate message. X and Y are a parameter of the signer set; this SIP does not fix them here (for example, 3 of 4). At least one matching certificate must come from a signer running client A, and at least one from a signer running client B. Today those clients are Anchor and go-ssv. This mitigates a shared reconstruction bug in one implementation, such as #972. When implementations disagree on the roots for a block, no checkpoint is published and the disagreement is investigated.

This rule makes the mechanism intentionally inert until at least two implementations independently compute the same canonical root and validate it against the shared conformance vectors of section 6 at a common block. When signer infrastructure and a second conforming implementation are ready is an onboarding matter (see Out of Scope).

**Swapping the signature scheme later.** The certificate message records `scheme_version` and `signer_set_id` so a future version can change the signature scheme or rotate the signer set without changing the certificate message format. This includes a move to a post-quantum signature such as ML-DSA. Post-quantum signatures are not mandated for v1. Agility needs a floor; see the downgrade discussion in Security Considerations.

**Dedicated keys.** The signer keys must be separate keys created only for signing checkpoints. They must not reuse an operator's existing key. Reusing operator keys saves no distribution step, because a bootstrapping node needs the signer public keys before it has any state, so those keys ship with the client or its configuration regardless. They must also use a distinct signing context (domain separation) so a checkpoint signature can never be mistaken for, or replayed as, any other kind of signature.

**5. Client Import and Verification Flow**

When a node imports a checkpoint it performs the following checks. A first syncing node, or a node whose last sync is older than the maximum acceptable age, trusts the state itself; it cannot verify completeness without folding the logs, which is the work it is trying to skip. The checks below are the cheap bindings it can still enforce.

1. Confirm the certificate message network_id and ssv_contract_address match the node's configured network and contract. Reject on mismatch. Reject if canonical_spec_version is one the node does not implement, since a root is only meaningful under the canonical state definition that produced it.
2. Confirm block B is finalized according to the node's consensus data source. Reject if finality cannot be established. A finalized block can no longer be reverted by a chain reorganization (reorg), so the state at B is permanent. Confirm block_hash matches the finalized block.
3. Verify the certificates: each is a valid signature over this certificate message under scheme_version, the signers belong to a signer_set_id the node currently trusts, scheme_version is at or above the node's minimum accepted version, there are at least X matching certificates over the same certificate message, and the agreeing set spans both client implementations.
4. Parse the received payloads. If the bundle includes an optional snapshot_digest, confirm it for download integrity, but do not treat it as a signed trust anchor.
5. Reconstruct the canonical record set and active encrypted share set from the payloads. Recompute `state_root` and `share_set_root` using their canonical serializations. Confirm both roots match the certificate message.
6. Freshness: reject if block B is older than the maximum acceptable age (see below).
7. Monotonicity: reject if B is not newer than the node's current state. A node never imports a checkpoint older than what it already has.
8. On success, populate storage from the snapshot, locating and decrypting the node's own shares from the active encrypted share set, and continue normal sync after block B.

**Freshness and monotonicity.** Signatures cannot stop replay of an old but valid checkpoint. The defenses are a maximum acceptable age, enforced by the importing node, a finality requirement on B, and the rule that a node never imports a checkpoint older than its current state. Monotonicity protects only a resync, where the node already has state to compare against. A first syncing node has no current state, so monotonicity gives it nothing; its protections are the maximum age, the finality requirement, and the X of Y threshold across implementations.

**6. Conformance and Testing**

The canonical spec is only useful if it is enforced. The following checks do that, and they would have caught #972.

**CI conformance vectors.** A conformance vector is a recorded slice of real chain history paired with the state roots the canonical spec says it must produce. The expected roots are derived from the canonical spec, not from any one client. The vector file ships in the repository. Every conforming client replays the slice and asserts that its computed roots equal the recorded ones. This is fully automated. It covers only the history built into the test. The v1 vector set must include cases for the #972 operator public key layouts, two operator ids with the same canonical public key bytes, ValidatorAdded rejection after nonce advancement, ValidatorRemoved omission, removed operators that remain referenced by active membership, and removed operators that become unreferenced and are omitted.

**Full replay audit run.** An audit implementation can start at the deployment block, fold all history, and emit at every sampling point (every N blocks) a tuple:

```text
(block_number, full_log_set_hash, state_root, share_set_root)
```

Generating the tuples is automatic for a client or materializer that has the required historical logs and encrypted share material. Comparing the streams is a separate step performed by a small diff tool. Starting from the deployment block catches a historical bug such as #972, which only appears once the affected event is processed. This is the audit path, not the normal checkpoint production path.

The `full_log_set_hash` in the tuple is essential to this comparison. Without it, two clients could disagree only because their RPC providers served different events: providers differ in indexing, retention, and gaps. The diff tool compares `full_log_set_hash` first. If the log hashes differ, it is a data source problem, not a reconstruction bug. Only matching log hashes with differing roots indicate a real divergence. `full_log_set_hash` is audit evidence and is not a certificate field.

**Checkpoint production run.** For a checkpoint with a parent, each signer emits:

```text
(parent_checkpoint_hash, block_number, delta_log_set_hash, state_root, share_set_root)
```

The diff tool first compares the parent checkpoint hash and `delta_log_set_hash`, then compares the roots. This is the normal production comparison for later checkpoints and it does not require logs before the parent checkpoint. For an initial checkpoint, signers compare `(block_number, state_root, share_set_root)` and may also compare optional full replay audit evidence when they have it.

**Honest limit and complementarity.** Agreement across clients proves that the two clients reconstructed the same state. It does not prove the state is correct; both could share a spec mistake. eth_call spot checks are complementary because they compare known records against contract storage outside both clients, while root agreement catches omitted records that eth_call cannot ask about. Neither check catches a shared mistake in the event signatures, topics, contract address, deployment block, parent checkpoint, or delta range. The full fold from the deployment block remains the audit path for that class of error.

**Open Questions and Future Versions**

The following questions are outside the v1 acceptance path. A v1 importer does not choose among these options; it follows the closed profile in sections 1 and 2.

- O1. Should the checkpoint payload be one combined file with state and shares, or separate payloads for public state and encrypted shares? This is a distribution and file format question only. Acceptance always verifies the mandatory `state_root` and `share_set_root` from the certificate message. Open.
- O2. A future canonical spec version may replace the flat hash with a Merkle tree for inclusion proofs or Candidate B. That version must pin the exact tree shape, including leaf grouping, domain separation for leaves and internal nodes, and odd leaf handling.
- O3. A future canonical spec version may include data derived from beacon state, such as validator index. V1 excludes it because it is not a deterministic function of the SSV logs.
- O4. Post-quantum signature scheme for v1 or later. V1 keeps this optional behind the versioned scheme_version. Open.
- O5. A future canonical spec version may revisit operator identity or public key normalization. V1 keys operators by `operator_id`, does not deduplicate by public key, and hashes the canonical public key bytes defined in section 1.
- O6. A future canonical spec version may revisit tombstone policy. V1 omits removed validators, retains removed operators only while active clusters reference them, and omits clusters with no active validators.

**Out of Scope**

The following are operational, not protocol, and are out of scope for this SIP: where checkpoints are hosted and how they are distributed, how signer parties are onboarded, the process for resolving disputes when implementations disagree, and the production cadence of how often checkpoints are generated. How a producer stores, indexes, or retains encrypted share material is also out of scope, as long as the producer can enumerate the canonical records required by this SIP. The maximum age a node will accept is a validation rule enforced by the node and is in scope, as specified in section 5.

**Security Considerations**  

**Trust assumption.** An importing node that skips the fold trusts the signer set, not whoever published the checkpoint payloads. The publisher is untrusted; a malicious publisher can at most serve payloads that fail to reconstruct the signed `state_root` or `share_set_root`. The acceptance rule in section 4 assumes enough honest signers to prevent a dishonest quorum from reaching X while spanning both implementations.

**Completeness versus correctness.** Agreement across implementations catches omitted records only because the root commits to the record set itself, not to counters. It does not prove the shared event scope or canonical spec is correct; section 6 describes why eth_call and the full fold audit remain complementary.

**Reconstruction bug risk.** Two signers running the same client share that client's reconstruction bugs and would sign the same wrong roots. The acceptance rule across implementations mitigates this by requiring at least one certificate from each implementation, and the canonical application semantics in section 1 reduce the space for legitimate root differences.

**Equivocation and signer dishonesty.** If enough signers collude to reach X and span both implementations, they can sign wrong roots that are internally consistent. Nothing in the format binds a block number B to a single certificate message, so a dishonest quorum or publisher could serve two well formed certificate messages for the same block B and partition nodes onto divergent state. Equivocation is detectable out of band but is not prevented by the format alone. A recommended mitigation is for signers to publish each (block_number, certificate message) into an append only transparency record, so two conflicting signatures over the same block are publicly attributable.

**Replay of stale checkpoints.** Signatures over an old but valid certificate message stay valid forever, so signatures alone cannot stop replay. Section 5 defines the defenses: maximum acceptable age, finality, and monotonicity where a node already has current state.

**Downgrade and version attacks.** Cryptographic agility introduces several valid scheme_version, signer_set_id, and schema_version values over time. Without a floor, that enables downgrade: after a post-quantum migration an attacker could replay a checkpoint certified under the old, now weak classical scheme; after key rotation an attacker could replay certificates from a retired signer set. The maximum age rule only partly helps, since an old certificate message under a weak scheme within the age window would still pass. The importing node must therefore enforce a minimum acceptable scheme_version and reject retired signer_set_ids, so a checkpoint signed under a deprecated scheme or a retired signer set is rejected even when its signatures are cryptographically valid and it is within the age window. Agility without a floor is a downgrade vector.

**Signer key compromise.** A compromised signing key can sign a wrong certificate message. The true threshold is governed by both X and the signer counts for each implementation, so those values must be sized together. Rotation is not instantaneous protection: a node keeps trusting a signer set until it receives a client or configuration update, so revocation latency is bounded by the client release and update cadence.

**Supply chain dependency.** Signer public keys ship with the client or its configuration, because a bootstrapping node needs them before it has any state. The real root of trust is therefore client release integrity: the release channel delivers both the importer logic and the signer public keys, so a compromised release could defeat every check across implementations at once. This adds no new trust surface beyond trusting the client binary, but it raises the requirements on the release channel: reproducible builds, signed releases, and pinned, auditable signer set ids.

**Establishing finality.** The block_hash and finality checks in import step 2 are only as trustworthy as the node's beacon and execution data source. A node trusting a malicious RPC for finality could be shown a chain that is not canonical where B and block_hash look consistent. This is an existing trust assumption inherited from normal operation, not one introduced here; a consensus light client that would remove it is out of scope. Normal v1 acceptance requires finality; a node that cannot establish finality rejects the checkpoint.

**Unsafe recovery mode.** An implementation may offer an operator controlled recovery mode that imports a checkpoint at a block that is not finalized but is buried under a locally configured depth. This mode is outside normal v1 acceptance, must be disabled by default, and must be presented as unsafe because a later reorg can invalidate the imported state. If that happens, the node must discard the checkpoint state and resync from a finalized checkpoint or from logs.

**Share set correctness.** A malicious publisher cannot omit, corrupt, or replace a joining operator's encrypted share without changing `share_set_root`. A joining node must still decrypt its own share and validate it against the share public key before relying on it, because the root proves the ciphertext and share public key match the certified state, not that the local operator can successfully use the decrypted result.

**Encrypted share exposure.** The checkpoint must never include plaintext validator shares. It includes encrypted share ciphertexts that are already public in ValidatorAdded event data and transaction calldata. This preserves the existing long term security assumption: the encryption to operator keys and the threshold share scheme must remain sound. If enough operator decryption keys are compromised for the same validator, the validator can be compromised whether the ciphertexts come from historical logs or from a checkpoint.

**Trustless fallback and its limit.** The signer layer accelerates startup; it does not replace independent reconstruction. Its limit is log availability: as old logs stop being retained by default, the audit increasingly depends on a node or provider that still has them.
