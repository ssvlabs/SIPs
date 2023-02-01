| Author      | Title                    | Category | Status   |
|-------------|--------------------------|----------|----------|
| Alon Muroch | Beacon Node Fork Support | Core     | open-for-discussion |

**Summary**  
An SSV node should be able to support beacon-node forks by support multiple versions of beacon objects.

**Versioned Beacon Objects**

Blocks  

| **Version**   |
|-----------|
| Phase0    |
| Altair    |
| Bellatrix |
| Capella   |

**SSV Beacon Object Signing**

| Phase          | Objects signed                                                                                                |
|----------------|---------------------------------------------------------------------------------------------------------------|
| pre-Consensus  | randao, selection proof, contribution proofs                                                                  |
| consensus      | --                                                                                                            |
| post-consensus | attestationData, beacon block, blinded beacon block, aggregation, sync committee, sync committee contribution |

**Specification**

All beacon node api calls will use an interface{} and DataVersion as inputs. Versioning will then be taken care of on the implementation level 

```go
type BeaconNode interface {
    // GetAttestationData returns attestation data by the given slot and committee index
    GetAttestationData(slot phase0.Slot, committeeIndex phase0.CommitteeIndex) (interface{}, spec.DataVersion, error)
    // SubmitAttestation submit the attestation to the node
    SubmitAttestation(attestation interface{}, version spec.DataVersion) error
    
    // GetBeaconBlock returns beacon block by the given slot and committee index
    GetBeaconBlock(slot phase0.Slot, committeeIndex phase0.CommitteeIndex, graffiti, randao []byte) (interface{}, spec.DataVersion, error)
    // GetBlindedBeaconBlock returns blinded beacon block by the given slot and committee index
    GetBlindedBeaconBlock(slot phase0.Slot, committeeIndex phase0.CommitteeIndex, graffiti, randao []byte) (interface{}, spec.DataVersion, error)
    // SubmitBeaconBlock submit the block to the node
    SubmitBeaconBlock(block interface{}, version spec.DataVersion) error
    // SubmitBlindedBeaconBlock submit the blinded block to the node
    SubmitBlindedBeaconBlock(block interface{}, version spec.DataVersion) error

    // SubmitAggregateSelectionProof returns an AggregateAndProof object
    SubmitAggregateSelectionProof(slot phase0.Slot, committeeIndex phase0.CommitteeIndex, committeeLength uint64, index phase0.ValidatorIndex, slotSig []byte) (interface{}, spec.DataVersion, error)
    // SubmitSignedAggregateSelectionProof broadcasts a signed aggregator msg
    SubmitSignedAggregateSelectionProof(msg interface{}, version spec.DataVersion) error

    // GetSyncMessageBlockRoot returns beacon block root for sync committee
    GetSyncMessageBlockRoot(slot phase0.Slot) (phase0.Root, error)
    // SubmitSyncMessage submits a signed sync committee msg
    SubmitSyncMessage(msg interface{}, version spec.DataVersion) error
    
    // GetSyncCommitteeContribution returns
    GetSyncCommitteeContribution(slot phase0.Slot, subnetID uint64) (interface{}, spec.DataVersion, error)
    // SubmitSignedContributionAndProof broadcasts to the network
    SubmitSignedContributionAndProof(contribution interface{}, version spec.DataVersion) error
}
```

Consenus data will carry the beacon objects as []byte, an additional DataVersion param will be added to help unmarshal objects correctly

```go
type ConsensusData struct {
    Duty                 Duty
	DataVersion         spec.DataVersion // github.com/attestantio/go-eth2-client/spec
    Data                []byte
}

func (cid *ConsensusData) GetAttestationData() (interface{}, error) {
    switch cid.DataVersion {
        case spec.DataVersionPhase0:
            ret := &phase0.AttestationData{}
            if err := ret.UnmarshalSSZ(cid.Data); err != nil {
                return nil, err
            }
            return ret, nil
        default:
            return nil, errors.New("unknown attestation data version")
    }
}

func (cid *ConsensusData) GetBlockData() (interface{}, error) {
    switch cid.DataVersion {
        case spec.DataVersionPhase0:
        case spec.DataVersionAltair:
        ...
    }
}

func (cid *ConsensusData) GetBlindedBlockData() (interface{}, error) {
    switch cid.DataVersion {
        case spec.DataVersionPhase0:
        case spec.DataVersionAltair:
        ...
    }
}
```

The rest of the code should treat beacon objects as interface{}, occassionally casting to ssz.HashRoot or ssz.Marshaler as needed 