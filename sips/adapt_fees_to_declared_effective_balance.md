| Author | Title          | Category                             | Status  | Date               |
|-------|----------------|--------------------------------------|---------|--------------------|
| Gal Rogozinski & Marco Tabasco | Adapt fees to declared effective balance | Contracts & Core | open-for-discussions  |18.09.2024 |


## Summary

Creates a mechanism to incentivize users to record the correct balance of their validator with the contract.
This balance will be used for operator and network fee calculations.

## Motivation

Operators on SSV Network charge validators a fee based on an expected workload. The expected workload is proportional to the the effective balance a validator has. Currently the effective balance is assumed to be a constant 32 ETH by the contract. With the introduction of [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) the effective balance of a validator could be as high as 2048 ETH.

The same reasoning applies for the network fee charged to each validator registered.


## Rationale

Do not perform duties if the validator's effective balance is greater by more than a deviation configured by the contract's owner. This will incentivize validators to always register the correct amount on the contract. Effective balance may grow due to compounding rewards and it is the user's responsibility to update their effective balance before their operators pause. Paused operators will resume when the effective balance is updated on the contract.

## Specification

### Contract - track maxeb per cluster

When registering new validators, add a new parameter to indicate the current effective balance of the total amount of validators being added. This will be used to update operators' [snapshots](https://github.com/ssvlabs/ssv-network/blob/583b7b7cb1c1abc5d4c3b13bafca59bf315113b6/contracts/interfaces/ISSVNetworkCore.sol#L10) with the right amounts, calculate the network fee applied to the cluster, and emit the event with this information to allow the node confirm it.

#### Adapted functions

```ts
function registerValidator(
    bytes calldata publicKey,
    uint64[] memory operatorIds,
    bytes calldata sharesData,
    uint256 clusterEB, // This is new
    uint256 amount,
    Cluster memory cluster
) external
```
```ts
function bulkRegisterValidator(
    bytes[] memory publicKeys,
    uint64[] memory operatorIds,
    bytes[] calldata sharesData,
    uint256 clusterEB, // This is new
    uint256 amount, // SSV
    Cluster memory cluster
) external
```


### New functions

```ts
// Function to update the effective balance for a cluster
function updateEthEffectiveBalance(
    uint64[] memory operatorIds,
    uint256 clusterEB, // check lower type
    Cluster memory cluster
) external
```
```ts
// Function to update the maximum deviation allowed for the ETH 
// balance declared by validator owners
// @dev Using 10000 to represent 2 digit precision: 100% = 10000, 10% = 1000
// @dev Only the owner of the contract can perform the update
function updateAllowedBalanceDeviation(uint64 percentage) external onlyOwner
```

```ts
// Function to get the maximum ETH balance deviation
function getAllowedBalanceDeviation() 
    external view returns(uint64 percentage)
```

### New events
```ts
event ebUpdated(
    address indexed owner,
    uint64[] operatorIds,
    uint256 clustereb);
```

```ts
event AllowedBalanceDeviationUpdated(uint64 percentage);
```

### Current implementation

Operator earnings formula since snapshot:


`(currentBlock - snapshotBlock) * operatorFee * operatorValidatorCount`

### New implementation

Operator earnings for last snpashot + fees for one block.

`((currentBlock - snapshotBlock) * (snapshotTotalETH / 32) * operatorFee)`

Where`snapshotTotalETH`: The total ETH declared by the validators managed by this operator.


#### Network fee
##### Current implementation
`(currentBlock - snapshotBlock) * networkFee * totalValidatorCount`

##### New implementation
`(currentBlock - snapshotBlock) * (snapshotTotalETH / 32) * networkFee)`


#### User should always report effective balance when depositing SSVs

```ts
function deposit(
    address clusterOwner,
    uint64[] calldata operatorIds,
    uint256 amount,
    uint256 clusterEB,
    Cluster memory cluster
) external
```

### Fee management migration
Each 32 ETH declared when registering new validators, removing, etc. can be taken as a fee unit of the current validators count for operators and network.

These are the data structures to manage operator and network fees:
```ts
/// @notice Represents an SSV operator
struct Operator {
    /// @dev The number of validators associated with this operator
    uint32 validatorCount;
    /// @dev The fee charged by the operator
    uint64 fee;
    ...
}

/// @notice The count of validators governed by the DAO
uint32 daoValidatorCount;
/// @notice The current network fee value
uint64 networkFee;
```

Using the formulas described before to calculate operator and network earnings, we can conclude that:

`(snapshotTotalETH / 32)` is a representation of the actual `Operator.validatorCount` and `daoValidatorCount`. So 32 ETH declared is 1 fee unit.

##### Register validators example
Instead of using the number of keys being registered to update operators' and network snapshots and cluster metadata, the result of `declared maxEB / 32` will be used to calculate the fee units.

##### Proposed naming changes
```ts
/// @notice Represents an SSV operator
struct Operator {
    /// @dev The number of fee units associated with this operator
    uint32 feeUnits;
    /// @dev The fee charged by the operator
    uint64 fee;
    ...
}

/// @notice The number of fee units governed by the DAO
uint32 daoFeeUnits;
/// @notice The current network fee value
uint64 networkFee;
```


### Node

Node should keep a table of the validator's effective balances reported by the contract's `AllowedDeviationUpdated` events.
This table is updated upon contract events reporting balance updates.

When fetching duties for a validator, query the Eth node for its balance in the `Finalized` state. If the difference between the true balance and the effective balance reported by the user is larger than the allowed deviation do not execute the duty.


```python
  def skipDuties(valPK ValidatorPublicKey, deviation_percentage float)
    true_balance = GET /eth/v1/beacon/states/finalized/validator_balances?valPK
    reported_eff_balance = getFromEvent(valPK)
    true_balance = min(2048 ETH, true_balance)
    # Skip duties if this statement hold
    return RoundDown(true_balance) - reported_eff_balance > deviation_percentage * reported_eff_balance
```

A committee consensus process should only be skipped if all paritciapting validators should be skipped.

Note: we use the actual balance and not effective balance since it is easy to query.

## Issues and alternatives

1. We do not track changing balance due to compounding... meaning users will pay a bit less then what they manage. So the deviation should be configured in a way that allow acceptable slippage while not degrading UX by forcing users to update theor balance too soon.
2. Are users incentivized to split validators? By splitting a 2048 eth validators to two validators, a validator enjoys compounding and pays a reduced fee.
3. There was a considered option to allow contract's owner to update the APR offered by Ethereum. Then estimate the correct fee in the presence of rewards compunding and allow a smaller balance deviation. This proposal favors simplicity.
4. Oracles can help calibrate the calculations of an exact fee... however using a 3rd party oracle will add more complexity and will probably be a paid service.