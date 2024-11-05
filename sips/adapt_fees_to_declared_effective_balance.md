| Author | Title          | Category                             | Status  | Date               |
|-------|----------------|--------------------------------------|---------|--------------------|
| Gal Rogozinski & Marco Tabasco | Adapt fees to declared effective balance | Contracts & Core | open-for-discussions  |18.09.2024 |


## Summary

Creates a mechanism to incentivize users to record the correct balance of their validator with the contract.
This balance will be used for operator and network fee calculations.

## Motivation

Operators on SSV Network charge validators a fee based on an expected workload. The expected workload is proportional to the effective balance a validator has. Currently, the effective balance is assumed to be a constant 32 ETH by the contract. With the introduction of [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) the effective balance of a validator could be as high as 2048 ETH.

The same reasoning applies to the network fee charged to each validator registered.


## Rationale

Do not perform duties if the validator's effective balance is greater by more than a deviation configured by the contract's owner. This will incentivize validators to always register the correct amount on the contract. Effective balance may grow due to compounding rewards and it is the user's responsibility to update their effective balance before their operators pause. Paused operators will resume when the effective balance is updated on the contract.

## Specification

### Contract - Operators

Currently, operators set a fee (denominated in SSV) for the services provided to validator owners. This fee is charged per validator per block.

An operator owner can change the fee of an operator following this [flow](https://docs.ssv.network/learn/operators/update-fee).

The operator earnings are calculated on a cumulative basis, updating the operator balance when these events occur:

- Operator removed
- Execute operator fee change
- Withdraw operator earnings
- Register validators
- Remove validators
- Liquidate cluster
- Reactivate cluster

To calculate the operators' earnings, the number of validators managed is used along with the fee. With the introduction of EIP-7251, this math becomes obsolete. The operators' fees should consider the validators' declared effective balance to be fair.

#### Summary of the changes
Validators now can deposit ETH in this range, with a precision of 1 Gwei (10^9 wei):

`ETH deposited = 2048 - 32`

An operator's fee now will be adjusted to handle the ETH effective balance declared by the validators it manages: the *fee per ETH managed model*. This fee is charged per block.

#### Adapted data structures

```c
struct Operator {
    uint32 validatorCount;
    uint64 ebManaged; // New property: effective balance managed
    uint64 fee; // Now represents the fee per ETH managed per block
    address owner;
    bool whitelisted;
    Snapshot snapshot;
}
```

#### Adapted formulas

Operator earnings formula since snapshot:

`current earnings = snapshotBalance + ((currentBlock - snapshotBlock) * fee * validatorCount)`

New formula:

`current earnings = snapshotBalance + ((currentBlock - snapshotBlock) * fee * ebManaged)`

#### Functions involving operator fees (no signature changes)
```ts
function registerOperator(
    bytes calldata publicKey,
    uint256 fee, // fee per ETH managed per block
    bool setPrivate
) external override returns (uint64 id)
```

```ts
function declareOperatorFee(
    uint64 operatorId, 
    uint256 fee // fee per ETH managed per block
) external
```

```ts
function reduceOperatorFee(
    uint64 operatorId,
    uint256 fee // fee per ETH managed per block
) external
```

#### Fee management migration

For all these cases, an ongoing migration process will be performed, meaning that there will not be an external process interacting with operators to change declared fees.

##### Case 1: Public operators with a declared fee and `validatorCount = 0`
In this case, when new validators use these operators:
```
1. Convert the current operator's fee value to the new fee per ETH managed. Taking into account the previous fee was declared for a 32 ETH deposited validators, the new fee can be calculated as:
new fee = current fee / 32
2. Store the effective balance declared by the validator registration and update the operator's snapshot
```
##### Case 2: Public operators with a declared fee and `validatorCount > 0`
For this case, when:
- Operator removed
- Declare operator fee change
- Withdraw operator earnings
- Register validators
- Remove validators
- Liquidate cluster
- Reactivate cluster

```
1. The operator's effective balance managed is calculated this way, taking into account the previous fee was declared for a 32 ETH deposited validators:

ebManaged = 32 ether * validatorCount

2. Convert the current operator's fee value to the new fee per ETH managed. The new fee can be calculated as:

new fee = current fee / 32

3. Only when doing validators actions: Store the effective balance declared and update the operator's snapshot
```

### Contract - Validators
Validator owners deposit SSV tokens to cover network and operators' fees. This mechanism will be kept but now the total effective balance for all validators in the cluster should be passed as a parameter in these cases:
- Register validators
- Remove validators
- Liquidate cluster
- Reactivate cluster
- Withdraw cluster balance

#### Adapted functions

When registering new validators, add a new parameter to indicate the current effective balance of the total amount of validators being added. This will be used to update operators' [snapshots](https://github.com/ssvlabs/ssv-network/blob/583b7b7cb1c1abc5d4c3b13bafca59bf315113b6/contracts/interfaces/ISSVNetworkCore.sol#L10) with the right amounts, calculate the network fee applied to the cluster, and emit the event with this information to allow the node to confirm it.

```ts
function registerValidator(
    bytes calldata publicKey,
    uint64[] memory operatorIds,
    bytes calldata sharesData,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    uint256 amount,
    Cluster memory cluster
) external

function bulkRegisterValidator(
    bytes[] memory publicKeys,
    uint64[] memory operatorIds,
    bytes[] calldata sharesData,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    uint256 amount, // SSV
    Cluster memory cluster
) external

function removeValidator(
    bytes calldata publicKey,
    uint64[] memory operatorIds,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external

function bulkRemoveValidator(
    bytes[] calldata publicKeys,
    uint64[] memory operatorIds,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external

function liquidate(
    address clusterOwner, 
    uint64[] calldata operatorIds,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external

function reactivate(
    uint64[] calldata operatorIds,
    uint256 amount,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external

function withdraw(
    uint64[] calldata operatorIds,
    uint256 amount,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external
```

#### New functions

```ts
// Function to update the effective balance for a cluster
function updateEthEffectiveBalance(
    uint64[] memory operatorIds,
    uint256 clusterEB, // declare ETH effective balance for all validators in the cluster
    Cluster memory cluster
) external

// Function to update the maximum deviation allowed for the ETH 
// balance declared by validator owners
// @dev Using 10000 to represent 2 digit precision: 100% = 10000, 10% = 1000
// @dev Only the owner of the contract can perform the update
function updateAllowedBalanceDeviation(uint64 percentage) external onlyOwner

// Function to get the maximum ETH balance deviation
function getAllowedBalanceDeviation() 
external view 
returns(uint64 percentage)
```

#### New events
```ts
event ebUpdated(
    address indexed owner,
    uint64[] operatorIds,
    uint256 clustereb);

event AllowedBalanceDeviationUpdated(uint64 percentage);
```

### Contract - Network fee

Similar to the fee management for operators, the network fee should be moved to the  *fee per ETH managed* model.

#### New data structures

```ts
/// @notice The total declared ETH by validators in the network
uint64 daoEbManaged;
```

Current formula to calculate network earnings:

`network earnings = daoBalance + ((currentBlock - daoIndexBlockNumber) * networkFee * daoValidatorCount)`

New formula:

`network earnings = daoBalance + ((currentBlock - daoIndexBlockNumber) * networkFee * daoEbManaged)`



### Node

Node should keep a table of the validator's effective balances reported by the contract's `ebUpdated` events.
This table is updated upon contract events reporting balance updates.

When fetching duties for a validator, query the Eth node for its cluster balance in the `Finalized` state. If the difference between the true balance and the effective balance reported by the user is larger than the allowed deviation do not execute the duty.


```python
  def skipDuties(valPK ValidatorPublicKey, deviation_percentage float)
    // can be found from validator added event
    ownerID, opIDs, valPks = getClusterAccordingToValidator(valPk)
    true_balances = GET /eth/v1/beacon/states/finalized/validator_balances?valPKs
    cluster_balance = sum(true_balances)
    //get from ebUpdated Event
    reported_cluster_balance = getReportedClusterBalance(ownerID, opIDs)
    eff_cluster_balance = min(2048 ETH*sizeof(valPks), cluster_balance)
    # Skip duties if this statement hold
    return RoundDown(eff_cluster_balance) - reported_cluster_balance > deviation_percentage * reported_cluster_balance
```

A committee consensus process should only be skipped if all participating validators should be skipped.

Note: we use the actual balance and not effective balance since it is easy to query.

## Issues and alternatives

1. We do not track changing balance due to compounding... meaning users will pay a bit less then what they manage. So the deviation should be configured in a way that allow acceptable slippage while not degrading UX by forcing users to update theor balance too soon.
2. Are users incentivized to split validators? By splitting a 2048 eth validators to two validators, a validator enjoys compounding and pays a reduced fee.
3. There was a considered option to allow contract's owner to update the APR offered by Ethereum. Then estimate the correct fee in the presence of rewards compunding and allow a smaller balance deviation. This proposal favors simplicity.
4. Oracles can help calibrate the calculations of an exact fee... however using a 3rd party oracle will add more complexity and will probably be a paid service.