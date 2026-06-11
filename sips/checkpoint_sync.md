| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Diego Marin Santos ([@diegomrsantos](https://github.com/diegomrsantos)) | Canonical State Root and Checkpoint Sync | Core | open-for-discussion | (none) | 2026-06-09 |

[Discussion](https://github.com/ssvlabs/SIPs/discussions/93)

**Summary**  

Every SSV node keeps a copy of the network state: the list of operators, validators, and clusters. Today a node builds that state by folding the SSV smart contract's event log from the deployment block.

On mainnet, that means replaying the full log history back to 2023. A checkpoint lets a node start from a recent finalized block, while full reconstruction remains the audit fallback for anyone who can still obtain the logs.

This SIP specifies the Candidate A path from the discussion: a signed checkpoint that requires no contract change.

A checkpoint bundle can parse cleanly while omitting records or describing them wrongly. V1 addresses that completeness problem by certifying a canonical `state_root` for public state and a `share_set_root` for active encrypted shares. The certificate binds those roots to a finalized block, network, SSV contract, canonical spec version, signer set, and signature scheme.

The SIP defines the canonical network state, checkpoint bundle format, certificate format, import verification flow, and conformance tests. Throughout this SIP, logs means the SSV smart contract's event log, served by full nodes over eth_getLogs.

Candidate B, which would publish the state root on the SSV contract, is stronger but requires a contract change and is not specified here.

**Rationale & Design Goals**  

Two independent implementations that fold the same logs should produce the same bytes. Today they do not always agree. Anchor issue [#972](https://github.com/sigp/anchor/issues/972) is a live example: Anchor and go-ssv decode an OperatorAdded operator public key differently, which causes Anchor to silently skip an operator and diverge from the other client.

The design goals follow from that observation.

- A node that skips the fold must still be protected against incomplete or misrepresented state by agreement on canonical roots, not by trust in the publisher.
- The canonical state must pin encoding, ordering, and application semantics so independent clients can compare roots meaningfully.
- Checkpoint sync should remove full replay from startup while preserving replay from the deployment block as the audit path.
- The format must allow signer rotation and later signature scheme changes without a bundle format break.
- The mechanism must commit at finalized blocks so a reorg cannot invalidate a published checkpoint.

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
| Validator | validator public key (BLS), owner address; the pair is the validator identity, cluster_id |
| Cluster | cluster_id (derived, see below), owner address, sorted member operator_ids, liquidated flag |
| Owner | owner address, fee_recipient, next_validator_nonce |

**Counters are not a substitute for records.** The canonical root must commit to the full record set by content. It must not substitute any monotonic counter, count, or high water mark for the records, because counters can agree while a client is missing a record.

**Application semantics are part of the canonical state.** The canonical state is defined by the encoding and ordering below plus the rules for accepting, ignoring, and rejecting events. A client must not substitute local storage behavior for these canonical fold rules.

**Owner state and validator nonces.** The canonical Owner record stores `next_validator_nonce`, the nonce value to use for the next ValidatorAdded by that owner. It does not store a client's local representation of the last consumed nonce, and it does not depend on whether a client stores that value as nullable. This distinguishes an owner that only changed its fee recipient, whose next validator nonce is still 0, from an owner that already consumed nonce 0.

An absent Owner record means the effective defaults: `fee_recipient = owner` and `next_validator_nonce = 0`. A FeeRecipientAddressUpdated event changes the effective `fee_recipient` and leaves `next_validator_nonce` unchanged, using 0 if no previous owner nonce state exists. A ValidatorAdded event is evaluated against the current `next_validator_nonce` for that owner and then advances `next_validator_nonce` by one before the rest of the event is validated. If the ValidatorAdded event is rejected after this point, the nonce advance remains and no validator record, cluster update, or encrypted share records are created. An Owner record is included in the canonical record set when either `fee_recipient != owner` or `next_validator_nonce != 0`; otherwise it is omitted because it is identical to the default state.

**Excluded from the state root.** V1 excludes data that is not derived from SSV logs or is not part of validator client membership:

- Beacon validator index is fetched from the beacon node, not derived from SSV logs.
- ValidatorExited is a beacon chain signal and does not change SSV registry records. A validator registry record is removed only by ValidatorRemoved.
- Operator fee and Cluster accounting fields carried by events are contract bookkeeping, not validator client membership state.
- Cluster liquidation status is derived from ClusterLiquidated and ClusterReactivated, not from the Cluster.active field. Those events update only an included cluster record. If no cluster record is included for that owner and operator set, the event changes no canonical record. A cluster created later starts with `liquidated = false` unless a later ClusterLiquidated event changes it.

**Removal representation.** V1 pins removal per entity. A ValidatorRemoved event removes the validator record identified by the event owner and validator public key from `state_root` and removes the encrypted share records for that same owner and validator public key from `share_set_root`. If that was the last active validator in a cluster, the cluster record is also omitted. An OperatorRemoved event sets `removed = true` for that operator while any active cluster still references it. A removed operator with no active cluster reference is omitted from the canonical record set. Active operators are included even if they are not members of an active cluster.

**Canonical encoding and ordering.** The following choices are pinned.

- Operator identity. Operators are keyed by `operator_id` only. The canonical fold must not deduplicate operators by raw event bytes, decoded RSA key bytes, PEM text, owner address, or any other key. If two OperatorAdded events assign two different operator ids but decode to the same RSA public key, v1 still has two operator records.
- Operator public key. OperatorAdded publicKey bytes have appeared in more than one layout, so v1 defines `canonical_public_key` explicitly. First, try to decode the field as the SSV operator public key wrapper, equivalent to ABI decoding one dynamic `bytes` value. If that succeeds and the decoded bytes parse as the expected base64 PEM RSA public key payload, those decoded bytes are the canonical bytes. If wrapper decoding fails, the raw event bytes are accepted only if they parse directly as the same base64 PEM RSA public key payload. Otherwise the OperatorAdded event is rejected as malformed. The canonical bytes are hashed exactly as bytes; clients must not reserialize the key into a different PEM, DER, JSON, or text layout before hashing. If a client cannot decode the key into this canonical form, it must stop or mark the event malformed according to the v1 fold, not silently skip the operator.
- Validator identity. Validators are keyed by the pair `(validator_public_key, owner)`, matching the contract registration key. The canonical fold must not deduplicate validators by validator public key alone. If two accepted ValidatorAdded events use the same validator public key with different owners, v1 includes two validator records, and their encrypted share records remain separate by owner.
- Cluster member set. A ValidatorAdded event uses the `operatorIds` order to pair share bytes with operators, as defined in section 2. After parsing, the cluster membership is represented as a set and serialized in ascending operator_id order.
- Addresses. All addresses are encoded as their 20 raw bytes when hashed. Any text rendering uses lowercase hex. EIP-55 checksum casing is not used in the hashed form.
- Fee recipient fallback. If no Owner record exists for an owner, the fee recipient defaults to the owner address.
- Owner nonce. Owner records serialize `next_validator_nonce`, the nonce value to use for the next ValidatorAdded by that owner. It starts at 0. Every ValidatorAdded log for the owner advances it exactly once before validation, including a malformed or rejected ValidatorAdded.
- BLS public key. Validator public keys are encoded as their raw compressed bytes when hashed.
- Root domain. The SSZ containers below include a fixed domain field, `canonical_spec_version = 1`, `network_id`, and `ssv_contract_address` from the certificate message. This prevents a state root from one network or SSV contract from being reused under another.

**Cluster id derivation.** The cluster id is derived, not stored, and must match across implementations:

```text
cluster_id = keccak256(
    owner_address_bytes (20 bytes)
    ++ for each operator_id in ascending sorted order:
        ( 24 zero bytes ++ operator_id as 8-byte big-endian )
)
```

**2. State Root Construction**

V1 uses SimpleSerialize (SSZ) serialization as the canonical byte encoding for root inputs and certificate messages. V1 roots are still computed with keccak256 over those SSZ bytes, rather than SSZ `hash_tree_root`, because inclusion proofs are not part of v1. A future canonical spec version may replace this with a tree root, but v1 accepts only the constructions below.

```text
state_root = keccak256(ssz_serialize(CanonicalStateV1))
share_set_root = keccak256(ssz_serialize(ShareSetV1))
certificate_message_hash = keccak256(ssz_serialize(CheckpointCertificateMessageV1))
delta_log_set_hash = keccak256(ssz_serialize(LogSetV1 with domain DELTA_LOG_SET_V1))
full_log_set_hash = keccak256(ssz_serialize(LogSetV1 with domain FULL_LOG_SET_V1))
```

SSZ container field order is the order shown in this section. The fixed domains are 32 byte values derived by right padding the ASCII labels `STATE_ROOT_V1`, `SHARE_SET_ROOT_V1`, `CHECKPOINT_CERT_V1`, `DELTA_LOG_SET_V1`, and `FULL_LOG_SET_V1` with zero bytes. `ssv_contract_address` and log addresses are `Vector[byte, 20]`. All roots, hashes, topics, and block hashes are `Bytes32`. All numeric fields are SSZ `uint64` unless this SIP explicitly gives a narrower or wider type. An all zero `Bytes32` value is exactly 32 zero bytes.

Record ordering for the root is fully defined so the SSZ serialization is deterministic. These orderings at the top level are normative for v1 and must be covered by the conformance tests in section 6.

- Operators are ordered by ascending operator_id.
- Validators are ordered by their canonical BLS public key bytes, then by owner address bytes; the owner comparison is the tie breaker when public keys match.
- Clusters are ordered by their derived cluster_id bytes.
- Owners are ordered by their address bytes.
- Within a cluster, the member operator_ids are in ascending order, as in the cluster_id derivation.

The v1 SSZ containers for the public state root are:

```text
CanonicalStateV1 = Container[
    domain: Bytes32,
    canonical_spec_version: uint64,
    network_id: uint64,
    ssv_contract_address: Vector[byte, 20],
    operators: List[OperatorRecordV1, MAX_OPERATORS],
    validators: List[ValidatorRecordV1, MAX_VALIDATORS],
    clusters: List[ClusterRecordV1, MAX_CLUSTERS],
    owners: List[OwnerRecordV1, MAX_OWNERS],
]

OperatorRecordV1 = Container[
    operator_id: uint64,
    canonical_public_key: List[byte, MAX_OPERATOR_PUBLIC_KEY_BYTES],
    owner: Vector[byte, 20],
    removed: boolean,
]

ValidatorRecordV1 = Container[
    validator_public_key: Vector[byte, 48],
    owner: Vector[byte, 20],
    cluster_id: Bytes32,
]

ClusterRecordV1 = Container[
    cluster_id: Bytes32,
    owner: Vector[byte, 20],
    operator_ids: List[uint64, MAX_CLUSTER_OPERATORS],
    liquidated: boolean,
]

OwnerRecordV1 = Container[
    owner: Vector[byte, 20],
    fee_recipient: Vector[byte, 20],
    next_validator_nonce: uint64,
]
```

`MAX_OPERATORS`, `MAX_VALIDATORS`, `MAX_CLUSTERS`, `MAX_OWNERS`, `MAX_OPERATOR_PUBLIC_KEY_BYTES`, and `MAX_CLUSTER_OPERATORS` are fixed SIP constants for `canonical_spec_version = 1`. They set checkpoint encoding limits and must be high enough to cover every state that can exist under the SSV contract. They must be identical across implementations. A client rejects a v1 state that cannot be represented within these limits. The concrete values are part of the v1 conformance vectors.

The v1 SSZ container for the active encrypted share set is:

```text
ShareSetV1 = Container[
    domain: Bytes32,
    canonical_spec_version: uint64,
    network_id: uint64,
    ssv_contract_address: Vector[byte, 20],
    shares: List[EncryptedShareRecordV1, MAX_ENCRYPTED_SHARE_RECORDS],
]

EncryptedShareRecordV1 = Container[
    owner: Vector[byte, 20],
    validator_public_key: Vector[byte, 48],
    cluster_id: Bytes32,
    operator_id: uint64,
    share_public_key: Vector[byte, 48],
    encrypted_share_bytes: Vector[byte, 256],
]
```

`MAX_ENCRYPTED_SHARE_RECORDS` is also a fixed SIP constant for v1. Encrypted share records are ordered by validator public key bytes, owner address bytes, and then by ascending operator_id.

A ValidatorAdded log derives encrypted share records from the event `shares` bytes using the v1 share blob grammar below. Let `n = len(operatorIds)`. The `operatorIds` array in a ValidatorAdded log must be nonempty and strictly ascending, and every operator id must refer to an operator record present in the fold before the event. Duplicate ids, descending ids, or missing operators make the ValidatorAdded event malformed after the owner nonce transition defined in section 1.

```text
ValidatorAddedSharesV1 =
    validator_signature: Vector[byte, 96]
    share_public_keys: Vector[byte, 48] * n
    encrypted_share_bytes: Vector[byte, 256] * n
```

The expected `shares` length is `96 + n * 48 + n * 256` bytes. The `share_public_keys` entry at index i and the `encrypted_share_bytes` entry at index i are assigned to the `operatorIds` entry at index i from the event. This event order mapping is used only to form records; the final `ShareSetV1.shares` list is then sorted by the v1 record ordering above. No Snappy decompression, ABI decode inside the `shares` value, JSON decode, or local database layout is part of this grammar.

The `validator_signature` is checked before any validator record, cluster update, or encrypted share record is created. The signature is a 96 byte BLS signature by `validator_public_key` over:

```text
validator_registration_message_v1 =
    keccak256(utf8(owner_eip55_hex ++ ":" ++ decimal_next_validator_nonce))
```

`owner_eip55_hex` is the EIP-55 address text with `0x` prefix derived from the event owner address. `decimal_next_validator_nonce` is the base 10 nonce text with no leading zero, except that zero is encoded as `0`. This text form is used only for the existing validator registration signature. Addresses in the SSZ records and roots still use raw 20 byte values.

After the owner nonce is read and advanced, a ValidatorAdded event is malformed if the validator public key is not a 48 byte BLS public key, the operator id array is invalid under the rules above, the `shares` length does not match the v1 formula, or `validator_signature` does not verify. A malformed ValidatorAdded event leaves the nonce advance in place and creates no validator record, cluster update, or encrypted share records.

V1 does not decrypt `encrypted_share_bytes`, verify plaintext shares, or check that a ciphertext matches its `share_public_key`. The canonical fold commits to the public ciphertext bytes from the accepted event. A client may later fail to decrypt its own encrypted share and decide that the share is unusable locally, but that local outcome is outside the canonical fold and must not change `state_root` or `share_set_root`.

The v1 SSZ container for log set hashes is:

```text
LogSetV1 = Container[
    domain: Bytes32,
    canonical_spec_version: uint64,
    network_id: uint64,
    ssv_contract_address: Vector[byte, 20],
    from_block_number: uint64,
    to_block_number: uint64,
    events: List[RelevantSsvLogV1, MAX_LOG_EVENTS],
]

RelevantSsvLogV1 = Container[
    block_number: uint64,
    block_hash: Bytes32,
    transaction_index: uint64,
    log_index: uint64,
    address: Vector[byte, 20],
    topics: List[Bytes32, MAX_LOG_TOPICS],
    data: List[byte, MAX_LOG_DATA_BYTES],
]
```

The log set range is inclusive. `from_block_number` and `to_block_number` are part of the hash preimage even when the `events` list is empty. The `events` list contains the raw Ethereum logs for the relevant SSV events from section 1, emitted by `ssv_contract_address`, ordered by ascending `(block_number, transaction_index, log_index)`. `address` must equal `ssv_contract_address`, and `topics` and `data` are the exact bytes returned by the execution data source for that log. `MAX_LOG_EVENTS`, `MAX_LOG_TOPICS`, and `MAX_LOG_DATA_BYTES` are fixed SIP constants for v1.

**3. Checkpoint Format**

A checkpoint distribution is untrusted. In v1 the mandatory interchange object is `CheckpointBundleV1`, which contains the certificate message, signature records, canonical public state, and active encrypted share set. The certificate signs canonical SSZ commitments to public state and active encrypted shares. The bundle carries the preimages needed by an importing node to rebuild those commitments without replaying old logs.

Every conforming v1 importer must accept and verify the SSZ bundle format below:

```text
CheckpointBundleV1 = Container[
    certificate_message: CheckpointCertificateMessageV1,
    signatures: List[CheckpointSignatureV1, MAX_CHECKPOINT_SIGNATURE_RECORDS],
    canonical_state: CanonicalStateV1,
    share_set: ShareSetV1,
]
```

`MAX_CHECKPOINT_SIGNATURE_RECORDS` is a fixed SIP constant for `scheme_version = 1`. The mandatory bundle is serialized with SSZ. Signature records are untrusted transport data until the importer verifies them against `certificate_message`. Other transport encodings may be offered, but they are not sufficient for v1 conformance unless they translate into the exact SSZ bytes of `CheckpointBundleV1` before any certificate, root, or storage decision. Compression, chunking, mirrors, and transport digests are outside the trust path.

**Bundle contents.** The bundle carries `CanonicalStateV1` and `ShareSetV1` at block B. These are the preimages for the signed `state_root` and `share_set_root`, and an importer populates storage only from those verified containers.

The bundle carries the full active encrypted share set because a fresh operator bootstrap needs its encrypted share and clients may not retain every ciphertext locally. The ciphertexts are public event data; plaintext shares are not part of the checkpoint.

`state_root` commits to the public validator client registry state. `share_set_root` commits to encrypted share records for validators active at block B. It excludes encrypted shares for validators removed before B because those shares are not needed to bootstrap the current state.

`share_set_root` uses `ShareSetV1` and `EncryptedShareRecordV1` from section 2. The share blob grammar defines the boundary of each encrypted share before it is inserted into `EncryptedShareRecordV1`. This root proves share completeness and byte integrity for bootstrap; it does not require signers to decrypt shares.

**Bundle validation.** A v1 importer validates `CheckpointBundleV1` as canonical data before any storage mutation. It parses exactly one `certificate_message`, one `canonical_state`, and one `share_set`, recomputes `state_root` and `share_set_root` from those SSZ containers, and rejects if either root differs from the certificate message. After the roots match, the importer still rejects the bundle if the canonical sets are not internally consistent:

- duplicate operator records by operator_id;
- duplicate validator records by `(validator_public_key, owner)`;
- duplicate cluster records by cluster_id;
- duplicate owner records by owner address;
- duplicate encrypted share records by `(validator_public_key, owner, operator_id)`;
- any validator whose cluster_id does not resolve to a cluster record;
- any validator whose owner differs from the referenced cluster owner;
- any cluster member operator_id that does not resolve to an operator record;
- any encrypted share whose `(validator_public_key, owner)` does not resolve to a validator record;
- any encrypted share whose cluster_id differs from the referenced validator's cluster_id;
- any encrypted share whose operator_id is not a member of the referenced cluster;
- any active validator that does not have exactly one encrypted share for each operator_id in its cluster.

These checks make extra shares for unknown validators, removed validators, and nonmember operators invalid. They also make conflicting duplicate records invalid even if one duplicate alone would hash to a valid root. A v1 importer must populate storage only from the verified `CanonicalStateV1` and `ShareSetV1` records in `CheckpointBundleV1`. It must not populate storage from transport metadata, side tables, file names, indexes, caches, or any alternate representation that is not part of those verified SSZ containers.

**Checkpoint production paths.** The protocol is defined by records and roots, not by a particular database. A producer may materialize records from retained state, an archive source, a historical materializer, or a parent checkpoint plus new logs through B.

For an initial checkpoint, where `parent_checkpoint_hash` is the all zero `Bytes32` value, the producer must enumerate the complete public state and active encrypted share set at B before computing the roots. A client that does not retain all active encrypted share ciphertexts must use a materializer, an archive source, or a previous checkpoint that already contains the share set.

For every checkpoint with a nonzero `parent_checkpoint_hash`, producers start from the parent checkpoint state and apply only the relevant SSV logs from the block after the parent through B. The certificate includes `delta_log_set_hash` for that range, computed from `LogSetV1` with domain `DELTA_LOG_SET_V1`, `from_block_number = parent.block_number + 1`, and `to_block_number = B`.

Checkpoint distribution may include an optional `snapshot_digest` outside `CheckpointBundleV1` for download integrity, caching, or mirror comparison. That digest is not a trust anchor; import step 1 treats it only as a transport check.

**Certificate message.** The certificate message is the small object signers sign. In v1 the object is the SSZ container `CheckpointCertificateMessageV1`, and signers sign `certificate_message_hash = keccak256(ssz_serialize(CheckpointCertificateMessageV1))` under the signature scheme identified by `scheme_version`.

| Field | Meaning |
| ----- | ------- |
| domain | Fixed domain `CHECKPOINT_CERT_V1` |
| schema_version | Version of the certificate message format |
| canonical_spec_version | Version of the canonical state definition: the included event set and the application and encoding semantics of section 1 that determine the root |
| network_id | Network and chain id the state belongs to |
| ssv_contract_address | Address of the SSV contract whose registry state is represented |
| block_number | Block B the state was taken at |
| block_hash | Hash of block B |
| parent_checkpoint_hash | `certificate_message_hash` of the parent checkpoint, or the all zero `Bytes32` value for an initial checkpoint |
| state_root | Canonical state root from section 2 |
| share_set_root | Canonical root of the active encrypted share set |
| delta_log_set_hash | `LogSetV1` hash of the exact ordered list of relevant SSV events consumed after the parent checkpoint, or the all zero `Bytes32` value for an initial checkpoint |
| signer_set_id | Identifier of the signer set that may certify this message |
| signer_set_hash | Hash of the trusted signer metadata for signer_set_id |
| scheme_version | Identifier of the signature scheme used by certificates |

```text
CheckpointCertificateMessageV1 = Container[
    domain: Bytes32,
    schema_version: uint64,
    canonical_spec_version: uint64,
    network_id: uint64,
    ssv_contract_address: Vector[byte, 20],
    block_number: uint64,
    block_hash: Bytes32,
    parent_checkpoint_hash: Bytes32,
    state_root: Bytes32,
    share_set_root: Bytes32,
    delta_log_set_hash: Bytes32,
    signer_set_id: uint64,
    signer_set_hash: Bytes32,
    scheme_version: uint64,
]
```

For a checkpoint with a parent, `delta_log_set_hash` binds the certificate message to the exact events consumed between the parent checkpoint and B, so that a later comparison or audit can confirm both parties read the same delta logs. For an initial checkpoint, `delta_log_set_hash` is the all zero `Bytes32` value. A producer may publish optional audit metadata that records a full historical log hash computed from `LogSetV1` with domain `FULL_LOG_SET_V1`, `from_block_number` equal to the SSV contract deployment block, and `to_block_number = B`, but importers do not require it and it is not part of the certificate message.

The canonical_spec_version is distinct from schema_version: schema_version versions the certificate message format, while canonical_spec_version versions the canonical state definition that determines the root. The root is meaningful only under the definition that produced it, so a future SSV contract upgrade that adds an event that changes state or changes application semantics is a new canonical state generation that bumps canonical_spec_version and requires regenerating the conformance vectors of section 6. An importer rejects a canonical_spec_version it does not implement (import step 1) rather than comparing roots across generations.

**4. Certificates**

A checkpoint carries a certificate message and one or more signature records. A signature record identifies the signer and carries a signature over `certificate_message_hash` by that signer.

```text
CheckpointSignatureV1 = Container[
    signer_id: uint64,
    signature: List[byte, MAX_CHECKPOINT_SIGNATURE_BYTES],
]
```

`MAX_CHECKPOINT_SIGNATURE_BYTES` is a fixed SIP constant for `scheme_version = 1`.

`scheme_version = 1` uses individual signature records. Any future aggregate signature scheme must still expose the signer ids covered by the aggregate before applying the same counting rules.

**Scheme version 1 signatures.** `scheme_version = 1` uses the same RSA public key format and RSA with SHA-256 signature procedure used for SSV operator messages.

For v1, `TrustedSignerV1.public_key` is the ASCII byte string of the SSV operator public key format: the base64 encoding of a PEM RSA public key, with the same header and footer normalization used for OperatorAdded `canonical_public_key`. The decoded key must parse as a 2048 bit RSA public key. The key format is reused; the signing key itself must still be dedicated to checkpoint signing.

For v1, `MAX_CHECKPOINT_SIGNATURE_BYTES = 256`, and every `CheckpointSignatureV1.signature` must contain exactly 256 bytes. A different length makes the signature record invalid and ignored for signer counting.

The signed message bytes are exactly the 32 bytes of `certificate_message_hash`. The v1 signature is RSASSA-PKCS1-v1_5 with SHA-256 over those bytes. Equivalently, an implementation signs and verifies the byte string `certificate_message_hash` with the same RSA and SHA-256 procedure used for existing SSV signed messages. No text encoding, hex encoding, EIP-191 wrapper, or additional prefix is applied.

An importer verifies a signature by resolving the signer id to an active `TrustedSignerV1`, decoding `public_key` under the v1 key rules above, and verifying the signature over the certificate message hash. If public key decoding fails, the signer is not usable. If signature verification fails, that signature record is ignored. A signer id is counted only once, even if the bundle contains several valid signature records for it.

**Trusted signer set.** `signer_set_id` is not trusted by itself. An importer resolves it to a locally trusted `TrustedSignerSetV1`, shipped with the client or operator configuration before checkpoint import. The certificate message also commits to `signer_set_hash = keccak256(ssz_serialize(TrustedSignerSetV1))`, so signer set rotation cannot reuse an id with different metadata without changing the signed message.

```text
TrustedSignerSetV1 = Container[
    signer_set_id: uint64,
    scheme_version: uint64,
    threshold: uint64,
    min_controller_count: uint64,
    implementation_quorums: List[ImplementationQuorumV1, MAX_CHECKPOINT_IMPLEMENTATIONS],
    signers: List[TrustedSignerV1, MAX_CHECKPOINT_SIGNERS],
]

ImplementationQuorumV1 = Container[
    implementation_id: uint64,
    min_signers: uint64,
]

TrustedSignerV1 = Container[
    signer_id: uint64,
    implementation_id: uint64,
    controller_id: uint64,
    public_key: List[byte, MAX_SIGNER_PUBLIC_KEY_BYTES],
    valid_from_block: uint64,
    valid_to_block: uint64,
]
```

`MAX_CHECKPOINT_IMPLEMENTATIONS`, `MAX_CHECKPOINT_SIGNERS`, and `MAX_SIGNER_PUBLIC_KEY_BYTES` are fixed SIP constants for `scheme_version = 1`. `implementation_quorums` are ordered by ascending implementation_id, and signers are ordered by ascending signer_id before computing `signer_set_hash`. Duplicate implementation ids, duplicate signer ids, duplicate public keys, `threshold = 0`, or an implementation quorum with `min_signers = 0` make the trusted signer set invalid. `valid_to_block = 0` means the signer has no configured retirement block. Otherwise, a signer is active for block B only when `valid_from_block <= B <= valid_to_block`. `threshold` is the minimum number of unique active signer ids required. `min_controller_count = 0` disables controller diversity; otherwise the counted signer set must contain at least that many distinct `controller_id` values.

V1 defines these implementation ids:

```text
1 = Anchor
2 = go-ssv
```

For v1, importers reject a `TrustedSignerSetV1` whose `implementation_quorums` do not include `(implementation_id = 1, min_signers = 1)` and `(implementation_id = 2, min_signers = 1)`. A future `scheme_version` may add more implementations or change these minimums, but `scheme_version = 1` keeps both minima.

**Required meaning.** A certificate attests that the signer independently materialized the canonical public state and active encrypted share set at block B, without trusting the bundle being signed, and computed the signed roots. For a child checkpoint, it also attests that the signer consumed the exact ordered event delta identified by `delta_log_set_hash`. A certificate is not an attestation that a downloaded checkpoint file parsed or that a hosting path is trustworthy.

**Acceptance rule.** A checkpoint is accepted only when one set of unique active signer ids signs the same certificate message and satisfies `TrustedSignerSetV1`: at least `threshold` unique signers, every required implementation quorum, and `min_controller_count` when nonzero. Duplicate signer ids count once. Unknown, retired, wrong scheme, and invalid signature records are ignored before checking the threshold rules. For v1, the counted set must include both Anchor and go-ssv signers.

This keeps the mechanism inert until at least two implementations independently compute the same roots and validate them against the conformance vectors.

**Swapping the signature scheme later.** The certificate message records `scheme_version`, `signer_set_id`, and `signer_set_hash` so a future version can change the signature scheme or rotate the signer set without ambiguity. Post-quantum signatures are not mandated for v1. Agility needs a floor; see Security Considerations.

**Dedicated keys.** Signers must use keys created only for checkpoint signing, not existing operator keys. Bootstrapping nodes need signer public keys before they have state, so those keys ship with the client or configuration regardless. Signatures must also use a distinct signing context.

**5. Client Import and Verification Flow**

When a node imports a checkpoint it performs the following checks. A first syncing node, or a node whose last sync is older than the maximum acceptable age, trusts the state itself; it cannot verify completeness without folding the logs, which is the work it is trying to skip. The checks below are the cheap bindings it can still enforce.

1. Parse exactly one `CheckpointBundleV1` from the received SSZ bytes. Reject malformed SSZ, trailing bytes, or any representation that cannot materialize that single container. If checkpoint distribution includes an optional snapshot_digest, confirm it for download integrity, but do not treat it as a signed trust anchor.
2. Confirm the certificate message network_id and ssv_contract_address match the node's configured network and contract. Reject on mismatch. Reject if canonical_spec_version is one the node does not implement, since a root is only meaningful under the canonical state definition that produced it.
3. Confirm block B is finalized according to the node's consensus data source. Reject if finality cannot be established. A finalized block can no longer be reverted by a chain reorganization (reorg), so the state at B is permanent. Confirm block_hash matches the finalized block.
4. Resolve `signer_set_id` to a locally trusted `TrustedSignerSetV1`. Reject if the set is unknown, if its `scheme_version` differs from the certificate message, if `scheme_version` is below the node's minimum accepted version, or if `keccak256(ssz_serialize(TrustedSignerSetV1))` differs from `signer_set_hash`. Verify each signature record against the public key for its signer_id and count each valid active signer_id at most once. Reject unless the counted set satisfies `threshold`, every configured implementation quorum, and `min_controller_count` when nonzero.
5. Validate `canonical_state` and `share_set` as the canonical record set and active encrypted share set. Recompute `state_root` and `share_set_root` from their SSZ serializations, confirm both roots match the certificate message, and enforce the bundle validation rules from section 3.
6. Freshness: reject if block B is older than the maximum acceptable age (see below).
7. Monotonicity: reject if B is not newer than the node's current state. A node never imports a checkpoint older than what it already has.
8. On success, populate storage only from the verified `canonical_state` and `share_set`, locating and decrypting the node's own shares from the active encrypted share set, and continue normal sync after block B.

**Freshness and monotonicity.** Signatures cannot stop replay of an old but valid checkpoint. The defenses are a maximum acceptable age, enforced by the importing node, a finality requirement on B, and the rule that a node never imports a checkpoint older than its current state. Monotonicity protects only a resync, where the node already has state to compare against. A first syncing node has no current state, so monotonicity gives it nothing; its protections are the maximum age, the finality requirement, and the trusted signer set threshold plus implementation quorums.

**6. Conformance and Testing**

The canonical spec is only useful if it is enforced. The following checks do that, and they would have caught #972.

**CI conformance vectors.** A conformance vector is a recorded slice of real chain history paired with the roots and import result the canonical spec says it must produce. The expected roots and expected import result are derived from the canonical spec, not from any one client. The vector file ships in the repository. Every conforming client replays the slice, computes the roots, parses the import vector, imports only the vector's bundle bytes, and asserts that its computed roots and import decision equal the recorded values. This is fully automated. It covers only the history built into the test.

V1 import conformance vectors use this mandatory SSZ wrapper:

```text
CheckpointImportVectorV1 = Container[
    case_id: List[byte, MAX_VECTOR_CASE_ID_BYTES],
    expected_state_root: Bytes32,
    expected_share_set_root: Bytes32,
    expected_result: uint64,
    trusted_signer_sets: List[TrustedSignerSetV1, MAX_VECTOR_SIGNER_SETS],
    bundle_bytes: List[byte, MAX_VECTOR_BUNDLE_BYTES],
    untrusted_side_data: List[byte, MAX_VECTOR_SIDE_DATA_BYTES],
]
```

`MAX_VECTOR_CASE_ID_BYTES`, `MAX_VECTOR_SIGNER_SETS`, `MAX_VECTOR_BUNDLE_BYTES`, and `MAX_VECTOR_SIDE_DATA_BYTES` are fixed test format constants for v1. `bundle_bytes` is the exact byte input passed to the importer. Accepted cases encode one valid `CheckpointBundleV1`. Rejection cases may contain malformed SSZ, duplicate records, records outside canonical membership, bad certificate fields, or bad signatures. `expected_result` uses these v1 codes:

```text
0 = ACCEPT
1 = REJECT_BAD_BUNDLE_SSZ
2 = REJECT_BAD_CERTIFICATE
3 = REJECT_UNKNOWN_SIGNER_SET
4 = REJECT_SIGNER_SET_HASH_MISMATCH
5 = REJECT_MISSING_IMPLEMENTATION_QUORUM
6 = REJECT_DUPLICATE_CANONICAL_RECORD
7 = REJECT_EXTRA_ENCRYPTED_SHARE
8 = REJECT_MISSING_ENCRYPTED_SHARE
9 = REJECT_NONMEMBER_ENCRYPTED_SHARE
10 = REJECT_CLUSTER_MISMATCH
11 = REJECT_UNTRUSTED_SIDE_DATA_USED
```

`untrusted_side_data` is never part of `CheckpointBundleV1`; it exists only to test that implementations do not populate storage from bytes outside the verified bundle. A conformance harness reports code 11 if an implementation reads or stores from `untrusted_side_data`; a production importer must never receive that field.

The v1 vector set must cover these categories:

- operator identity and key decoding: #972 operator public key layouts, two operator ids with the same canonical public key bytes, and the same validator public key registered by two different owners;
- ValidatorAdded parsing and validation: nonce advancement before rejection, 4 and 7 operator committees, unsorted or duplicate operator ids, bad share blob length, bad validator signature, and ciphertext bytes that are structurally valid but do not decrypt for one operator;
- removal representation: ValidatorRemoved omission, removed operators that remain referenced by active membership, and removed operators that become unreferenced and are omitted;
- bundle consistency: malformed `bundle_bytes`, duplicate canonical records, duplicate encrypted share records, extra shares for unknown or removed validators, shares for nonmember operators, cluster_id mismatch, missing shares for cluster members, and conflicting duplicate entries;
- storage safety: bundle bytes with correct roots plus untrusted side data that must not be written to storage;
- certificate acceptance: valid v1 RSA signatures, invalid signer public key encoding, wrong signature length, signatures over the wrong message bytes, duplicate signer ids, missing implementation quorum, retired signer, signer_set_hash mismatch, and controller diversity when enabled.

**Production and audit evidence.** An audit implementation can start at the deployment block, fold all history, and emit at every sampling point:

```text
(block_number, full_log_set_hash, state_root, share_set_root)
```

The diff tool compares `full_log_set_hash` first. If log hashes differ, clients read different events. Only matching log hashes with differing roots indicate a reconstruction divergence. `full_log_set_hash` is audit evidence and is not a certificate field.

For a checkpoint with a parent, each signer emits:

```text
(parent_checkpoint_hash, block_number, delta_log_set_hash, state_root, share_set_root)
```

The diff tool compares the parent checkpoint hash and `delta_log_set_hash`, then the roots. This is the normal production comparison for later checkpoints and does not require logs before the parent checkpoint. For an initial checkpoint, signers compare `(block_number, state_root, share_set_root)` and may also compare optional full replay audit evidence when they have it.

**Honest limit and complementarity.** Agreement across clients proves that the two clients reconstructed the same state. It does not prove the state is correct; both could share a spec mistake. eth_call spot checks are complementary because they compare known records against contract storage outside both clients, while root agreement catches omitted records that eth_call cannot ask about. Neither check catches a shared mistake in the event signatures, topics, contract address, deployment block, parent checkpoint, or delta range. The full fold from the deployment block remains the audit path for that class of error.

**Open Questions and Future Versions**

The following questions are outside the v1 acceptance path. A v1 importer does not choose among these options; it follows the closed profile in sections 1 through 5.

- O1. A future canonical spec version may replace the flat hash with a Merkle tree for inclusion proofs or Candidate B. That version must pin the exact tree shape, including leaf grouping, domain separation for leaves and internal nodes, and odd leaf handling.
- O2. A future canonical spec version may include data derived from beacon state, such as validator index. V1 excludes it because it is not a deterministic function of the SSV logs.
- O3. Post-quantum signature scheme for v1 or later. V1 keeps this optional behind the versioned scheme_version. Open.
- O4. A future canonical spec version may revisit operator identity or public key normalization. V1 keys operators by `operator_id`, does not deduplicate by public key, and hashes the canonical public key bytes defined in section 1.
- O5. A future canonical spec version may revisit tombstone policy. V1 omits removed validators, retains removed operators only while active clusters reference them, and omits clusters with no active validators.

**Out of Scope**

The following are operational, not protocol, and are out of scope for this SIP: where checkpoints are hosted and how they are distributed, how signer parties are onboarded, the process for resolving disputes when implementations disagree, and the production cadence of how often checkpoints are generated. How a producer stores, indexes, or retains encrypted share material is also out of scope, as long as the producer can enumerate the canonical records required by this SIP. The maximum age a node will accept is a validation rule enforced by the node and is in scope, as specified in section 5.

**Security Considerations**  

**Trust assumption.** An importing node that skips the fold trusts the signer set, not the checkpoint publisher. The publisher is untrusted and can at most serve a bundle that fails to reconstruct the signed roots.

**Completeness versus correctness.** Agreement across implementations catches omitted records only because the root commits to records, not counters. It does not prove the shared event scope or canonical spec is correct; eth_call spot checks and the full fold audit remain complementary.

**Reconstruction bug risk.** Signers running the same client share that client's reconstruction bugs. Implementation quorums mitigate this by requiring the counted signer set to include independent client implementations.

**Equivocation and signer dishonesty.** A dishonest quorum can sign wrong roots or two well formed certificate messages for the same block B. Equivocation is detectable out of band but not prevented by the format. A recommended mitigation is an append only transparency record for each `(block_number, certificate message)`.

**Replay and downgrade.** Old signatures remain valid, and cryptographic agility creates downgrade risk. Importers mitigate this with maximum acceptable age, finality, monotonicity when local state exists, a minimum `scheme_version`, and rejection of retired signer sets.

**Signer key compromise.** A compromised signer key can sign a wrong certificate message. The threshold, implementation quorums, and controller diversity rule must be sized together. Revocation latency is bounded by client release and configuration update cadence.

**Supply chain dependency.** Signer public keys and metadata ship with the client or configuration because a bootstrapping node needs them before it has state. The release channel therefore delivers both importer logic and trusted signer sets, so reproducible builds, signed releases, and auditable signer set hashes matter.

**Establishing finality.** The block_hash and finality checks are only as trustworthy as the node's beacon and execution data sources. This is inherited from normal operation. Normal v1 acceptance requires finality; a node that cannot establish finality rejects the checkpoint.

**Unsafe recovery mode.** An implementation may offer an operator controlled recovery mode that imports a checkpoint before finality using a local depth setting. This mode is outside normal v1 acceptance, must be disabled by default, and must discard checkpoint state if a later reorg invalidates it.

**Encrypted shares.** A malicious publisher cannot omit, corrupt, or replace a joining operator's encrypted share without changing `share_set_root`. A joining node must still decrypt its own share and validate it before relying on it. The checkpoint must never include plaintext validator shares.

**Trustless fallback and its limit.** The signer layer accelerates startup; it does not replace independent reconstruction. Its limit is log availability: as old logs stop being retained by default, the audit increasingly depends on a node or provider that still has them.
