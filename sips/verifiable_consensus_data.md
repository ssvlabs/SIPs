| Author         | Title        | Category | Status   |
| -------------- | ------------ | -------- | -------- |
| Gal Rogozinski | Verifiable Attestation | Core     | open-for-discussion |

## Summary

Add a `SignedBeaconBlockHeader` to attestation consensus data. Enable a consensus data value check that will ensure the attestation or sync committee  data will be accepted by the beacon chain.

## Rationale

Currently the consensus leader is trusted for creating valid, timely data for the latest state of the beacon chain. By making the data verifiable the trust assumption can be removed. 

This will open a pathway to future leaderless designs where the committee can pick the best value. 

## Design

The design is best achievable via a hard fork.
Introduce a new `VerifiableConsensusData` type.

```go
type VerifiableConsensusData struct {
    Duty    spec.Duty
    Version spec.DataVersion
    Proof   []phase0.SignedBeaconBlockHeader `ssz-max:"3"`
    DataSSZ []byte `ssz-max:"128"` // attestation data max size
}
```

All Attestation and SyncCommittee logic should be moved to work with the new type.

```go
func (cd VerifiableConsensusData) GetAttestationData (att phase0.AttestationData,
                                                     headProof phase0.SignedBeaconBlockHeader, 
                                                     targetProof phase0.SignedBeaconBlockHeader, 
                                                     sourceProof phase0.SignedBeaconBlockHeader, 
                                                     error) {
    if err := att.UnmarshalSSZ(cd.DataSSZ); err != nil {
        return nil, nil, nil, nil, errors.Wrap(err, "could not unmarshal attestation ssz")
    }
    // Head vote proof
    if err := headProof.UnmarshalSSZ(cd.Proof[0]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal head vote proof ssz")
    }
    // Target proof
    if err := targetProof.UnmarshalSSZ(cd.Proof[1]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal target proof ssz")
    }
    // Target proof
    if err := sourceProof.UnmarshalSSZ(cd.Proof[1]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal source proof ssz")
    }
    return att, headProof, targetProof, sourceProof, nil
}

func (cd VerifiableConsensusData) GetSyncCommitteeBlockRoot (phase0.Root, phase0.SignedBeaconBlockHeader, error) {
	ret := SSZ32Bytes{}
	if err := ret.UnmarshalSSZ(cd.DataSSZ); err != nil {
		return nil, errors.Wrap(err, "could not unmarshal ssz")
	}
    proof := phase0.SignedBeaconBlockHeader
    if err := proof.UnmarshalSSZ(cd.Proof); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal proof ssz")
    }
	return phase0.Root(ret), proof, nil
}

func (cd VerifiableConsensusData) Validate error {
    switch cid.Duty.Type {
	case BNRoleAttester:
            if _, _, _, _, err := cid.GetAttestationData(); err != nil {
                return err
            }
            return nil
	case BNRoleSyncCommittee:
            if _, _, err := cid.GetSyncCommitteeBlockRoot(); err != nil {
                return err
            }
            return nil
        default:
		return errors.New("unknown duty role")
	}
}
```

In addition we do the following changes to value checks mechanisms:
```go 
func verifyData(expectedRoot phase0.Root, proof phase0.SignedBeaconBlockHeader) error {
    beaconRoot := proof.Message.HashRoot()
    if !bytes.Equal(expectedRoot, beaconRoot) {
        return errors.New("unexpected header root")
    }
    blockEpoch := getBeaconNode().getEpochBySlot(proof.Message.slot)
    domain := GetBeaconNode().DomainData(blockEpoch, spec.DomainProposer)
    root := spec.ComputeEthSigningRoot(beaconRoot, domain)
    validator := GetBeaconNode.GetValidator("HEAD", proof.Message.ProposerIndex)
    return bls.Verify(sig, validator.PubKey, root)
}

func AttesterValueCheckF(
	signer types.BeaconSigner,
	network types.BeaconNetwork,
	validatorPK types.ValidatorPK,
	validatorIndex phase0.ValidatorIndex,
	sharePublicKey []byte,
    // obtained from the operator's beacon node
    attestationDataByts []byte,
) qbft.ProposedValueCheckF {
	return func(data []byte) error {
        // addition
        if bytes.Equal(attestationDataByts, data) {
            return nil
        }

		cd := types.VerifyableConsensusData{}
		if err := cd.Decode(data); err != nil {
			return errors.Wrap(err, "failed decoding consensus data")
		}
		if err := cd.Validate(); err != nil {
			return errors.Wrap(err, "invalid value")
		}

		if err := dutyValueCheck(&cd.Duty, network, types.BNRoleAttester, validatorPK, validatorIndex); err != nil {
			return errors.Wrap(err, "duty invalid")
		}

        // change
		attestationData, headProof, targetProof, sourceProof, err := cd.GetAttestationData() // error checked in cd.validate()
        

		if cd.Duty.Slot != attestationData.Slot {
			return errors.New("attestation data slot != duty slot")
		}

		if cd.Duty.CommitteeIndex != attestationData.Index {
			return errors.New("attestation data CommitteeIndex != duty CommitteeIndex")
		}

		if attestationData.Target.Epoch < network.EstimatedCurrentEpoch()-1 {
			return errors.New("attestation data target epoch is into far past")
		}
        
        //Addition
        if attestationData.Target.Epoch != getBeaconNode().EpochFromSlot(targetProof.Message.Slot) {
            return errors.New("Target epoch doesn't match proof")
        }
        

		if attestationData.Source.Epoch >= attestationData.Target.Epoch {
			return errors.New("attestation data source > target")
		}
        
        // Addition
        if attestationData.Source.Epoch != getBeaconNode().EpochFromSlot(sourceProof.Message.Slot) {
            return errors.New("Target epoch doesn't match proof")
        }
        

        if err := signer.IsAttestationSlashable(sharePublicKey, attestationData); err != nil {
            return err
        }
        
        // Addition
        // heavy checks should be in the end
        // can be optimized with batch verification
        if err := verifyData(attestationData.BeaconBlockRoot, headProof); err != nil {
            return errors.Wrap("invalid head vote proof", err)
        }
        
        if err := verifyData(attestationData.Target.Root, targetProof); err != nil {
            return error.Wrap("invalid target vote proof", err)
        } 
        
        if err := verifyData(attestationData.source.Root, sourceProof); err != nil {
            return error.Wrap("invalid target vote proof", err)
        } 
        
        return nil
	}
    
    func SyncCommitteeValueCheckF(
	signer types.BeaconSigner,
	network types.BeaconNetwork,
	validatorPK types.ValidatorPK,
	validatorIndex phase0.ValidatorIndex,
    // obtained from the operator's beacon node
    syncCommitteeBlockRoot []byte,
) qbft.ProposedValueCheckF {
	return func(data []byte) error {
        // addition
        if bytes.Equal(syncCommitteeBlockRoot, data) {
            return nil
        }

		cd := types.VerifiableConsensusData{}
		if err := cd.Decode(data); err != nil {
			return errors.Wrap(err, "failed decoding consensus data")
		}
		if err := cd.Validate(); err != nil {
			return errors.Wrap(err, "invalid value")
		}

		if err := dutyValueCheck(&cd.Duty, network, types.BNRoleSyncCommittee, validatorPK, validatorIndex); err != nil {
			return errors.Wrap(err, "duty invalid")
		}
        
        root, proof, _, cd.GetSyncCommitteeBlockRoot \\ checked on validate
        
        if err := verifyData(root, headProof); err != nil {
            return errors.Wrap("invalid head vote proof", err)
        }	
    }
}
    
```

In addition currently value checks are being performed in 2 places:
1. Upon starting a new instance
2. Upon validating received`qbft.Proposal` messages.

For all beacon duties besides block proposal, it can be assumed the source of the proposed consensus data (beacon node) is trusted by the operator and thus the first check (upon starting a new instance).

## Drawbacks

Currently this adds extra BLS checks. It is manadatory to alleviate signature verification load before proceeding.


## Open Questions

1. Do operators trust their beacon node? Maybe checks shouldn't be removed? Note that even in the presence of malicious beacon nodes, doing the value check on message processing should be enough.
2. It is possible to reject a value if it is deemed not timely by the node.. To try to force the round changes? Is it wise to do it now? Or should we wait for leaderless consensus.
    




