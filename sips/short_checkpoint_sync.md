| Author | Title | Category | Status | Dependency SIP | Date |
| ------ | ----- | -------- | ------ | -------------- | ---- |
| Diego Marin Santos ([@diegomrsantos](https://github.com/diegomrsantos)) | Canonical State Root and Checkpoint Sync - Short Spec | Core | open-for-discussion | (none) | 2026-06-09 |

[Discussion](https://github.com/ssvlabs/SIPs/discussions/93)

**Summary**

Define a signed checkpoint format that lets a node import recent SSV registry state without replaying all historical SSV contract logs. The checkpoint commits to a finalized block, a canonical public state root, and an active encrypted share set root.

**Motivation**

SSV nodes currently reconstruct registry state by replaying contract logs from deployment. This becomes slower and more fragile as history grows and old logs become harder to access. Checkpoint sync gives new or recovering nodes a bounded startup path while keeping full replay as the audit fallback.

The checkpoint must still prove completeness of the imported state. A bundle that merely parses is not enough: it could omit operators, validators, clusters, or encrypted shares. Therefore importers accept only canonical roots signed by a trusted threshold of independent signers and bound to a finalized block.

**Specification**

**Canonical State**

A v1 checkpoint is valid only for `canonical_spec_version = 1`.

To compute canonical state at block `B`, apply all relevant SSV contract logs from deployment through finalized block `B`, in `(block_number, transaction_index, log_index)` order.

Relevant events:

- `OperatorAdded`
- `OperatorRemoved`
- `ValidatorAdded`
- `ValidatorRemoved`
- `ClusterLiquidated`
- `ClusterReactivated`
- `FeeRecipientAddressUpdated`

`ValidatorExited` MUST NOT change checkpoint state.

The canonical public state MUST include:

- operators: `operator_id`, canonical public key, owner, removed status;
- validators: validator public key, owner, cluster id;
- clusters: cluster id, owner, sorted member operator ids, liquidation status;
- owners: owner address, fee recipient, next validator nonce.

The active encrypted share set MUST include one encrypted share record per active validator and cluster operator:

- owner;
- validator public key;
- cluster id;
- operator id;
- share public key;
- encrypted share bytes.

Removed validators and their encrypted shares MUST be omitted. Removed operators MUST be kept only while still referenced by an active cluster.

**Canonical Encoding**

V1 MUST use SSZ serialization for canonical containers and `keccak256(ssz_serialize(container))` for roots.

The checkpoint MUST define fixed SSZ limits for all lists. A client MUST reject data that cannot fit those limits.

Ordering MUST be deterministic:

- operators by ascending `operator_id`;
- validators by validator public key, then owner;
- clusters by `cluster_id`;
- owners by owner address;
- encrypted shares by validator public key, then owner, then `operator_id`;
- cluster members by ascending `operator_id`.

The state root preimage MUST include:

- fixed domain `STATE_ROOT_V1`;
- `canonical_spec_version`;
- `network_id`;
- `ssv_contract_address`;
- canonical operators, validators, clusters, and owners.

The share set root preimage MUST include:

- fixed domain `SHARE_SET_ROOT_V1`;
- `canonical_spec_version`;
- `network_id`;
- `ssv_contract_address`;
- active encrypted share records.

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

**Checkpoint Bundle**

The mandatory v1 interchange object is `CheckpointBundleV1`:

```text
CheckpointBundleV1 = Container[
    certificate_message: CheckpointCertificateMessageV1,
    signatures: List[CheckpointSignatureV1, MAX_CHECKPOINT_SIGNATURE_RECORDS],
    canonical_state: CanonicalStateV1,
    share_set: ShareSetV1,
]
```

Importers MUST verify only this SSZ bundle. Transport metadata, file names, side tables, indexes, mirrors, compression digests, and caches MUST NOT be trusted or imported as state.

**Certificate Message**

Signers sign:

```text
certificate_message_hash = keccak256(ssz_serialize(CheckpointCertificateMessageV1))
```

The certificate message MUST include:

- fixed domain `CHECKPOINT_CERT_V1`;
- schema version;
- canonical spec version;
- network id;
- SSV contract address;
- finalized block number;
- finalized block hash;
- parent checkpoint hash, or zero for an initial checkpoint;
- `state_root`;
- `share_set_root`;
- delta log set hash, or zero for an initial checkpoint;
- signer set id;
- signer set hash;
- signature scheme version.

For a child checkpoint, `delta_log_set_hash` MUST commit to the exact ordered relevant SSV logs from `parent.block_number + 1` through `B`.

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

**Signatures**

A checkpoint is accepted only if enough unique active trusted signers sign the same `certificate_message_hash`.

V1 signature rules:

- use dedicated checkpoint signing keys;
- use the SSV RSA public key format;
- sign the 32 byte `certificate_message_hash`;
- use RSASSA-PKCS1-v1_5 with SHA-256;
- count each signer id at most once;
- ignore unknown, inactive, duplicate, malformed, wrong-scheme, or invalid signatures.

The locally trusted signer set MUST define:

- signer set id;
- scheme version;
- threshold;
- optional minimum controller count;
- implementation quorum requirements;
- signer ids, implementation ids, controller ids, public keys, and validity block ranges.

The certificate message `signer_set_hash` MUST equal `keccak256(ssz_serialize(TrustedSignerSetV1))`.

For v1, the accepted signer set MUST include at least one Anchor signer and at least one go-ssv signer.

```text
CheckpointSignatureV1 = Container[
    signer_id: uint64,
    signature: List[byte, MAX_CHECKPOINT_SIGNATURE_BYTES],
]

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

**Import Flow**

A node importing a checkpoint MUST:

1. Parse exactly one SSZ `CheckpointBundleV1`; reject malformed SSZ or trailing bytes.
2. Reject unsupported `canonical_spec_version` or `scheme_version`.
3. Verify `network_id` and `ssv_contract_address` match local configuration.
4. Verify `block_hash` belongs to finalized block `B`.
5. Resolve the trusted signer set locally and verify `signer_set_hash`.
6. Verify certificate signatures and signer threshold rules.
7. Recompute `state_root` and `share_set_root` from the bundle contents.
8. Reject if either recomputed root differs from the certificate message.
9. Reject duplicate or inconsistent canonical records.
10. Reject missing, extra, duplicate, nonmember, or cluster-mismatched encrypted shares.
11. Reject checkpoints older than the node's maximum accepted age.
12. Reject checkpoints at or before the node's current local state height.
13. Populate storage only from verified `canonical_state` and `share_set`.
14. Decrypt and validate the node's own share before using it.
15. Continue normal sync from block `B + 1`.

**Bundle Consistency Rules**

An importer MUST reject:

- duplicate operators by `operator_id`;
- duplicate validators by `(validator_public_key, owner)`;
- duplicate clusters by `cluster_id`;
- duplicate owners by owner address;
- duplicate encrypted shares by `(validator_public_key, owner, operator_id)`;
- validators referencing missing clusters;
- validators whose owner differs from the cluster owner;
- clusters referencing missing operators;
- shares referencing missing validators;
- shares whose cluster id differs from the validator cluster id;
- shares for operators outside the validator's cluster;
- active validators without exactly one encrypted share per cluster operator.

**Fallback**

If checkpoint import fails, the node MUST NOT partially import checkpoint state. It MAY fall back to full log replay or another locally configured sync path.

**Tests**

Implementations SHOULD include conformance vectors for:

- canonical event folding and deterministic roots;
- operator public key decoding differences;
- validator identity by `(validator_public_key, owner)`;
- owner nonce advancement, including rejected `ValidatorAdded` events;
- removed validator, removed operator, and liquidated cluster behavior;
- valid and invalid bundle SSZ;
- root mismatch;
- duplicate and inconsistent canonical records;
- missing, extra, duplicate, nonmember, and mismatched encrypted shares;
- valid and invalid signer sets;
- invalid signatures, duplicate signer ids, retired signers, and missing implementation quorum;
- monotonicity and maximum age rejection.

**Security Requirements**

The checkpoint publisher is untrusted. Trust is only in finalized chain data, local importer rules, local trusted signer metadata, and valid threshold signatures over the certificate message.

Checkpoint sync MUST NOT include plaintext validator shares.

A checkpoint certificate proves agreement on the signed roots. It does not prove the canonical spec is correct, that all signers are honest, or that full historical audit is unnecessary.
