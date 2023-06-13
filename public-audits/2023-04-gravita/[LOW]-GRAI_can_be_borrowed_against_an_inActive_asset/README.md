# Original link
https://github.com/Gravita-Protocol/Gravita-SmartContracts/issues/314



# Vulnerability details

GRAI tokens can be borrowed against `inActive` collateral assets as the `getIsActive` check is not performed on `addCollateral` and `withdrawDebtTokens` functions.

## Affected smart contract

- BorrowerOperations
```solidity
 function addColl( 
 	address _asset, 
 	uint256 _assetSent, 
 	address _upperHint, 
 	address _lowerHint 
 ) external override { 
 	_adjustVessel(_asset, _assetSent, msg.sender, 0, 0, false, _upperHint, _lowerHint); 
 } 
```

## Description
The `AdminContract` contract has a `CollateralParams.active` flag which depicts the current active state of a collateral. This parameter can be changed using the setActive function.

The active state of a collateral is checked by querying the `AdminContract.getIsActive` function on all `BorrowerOperations.openVessel` calls. Hence the Gravita protocol intends to do not mint GRAi tokens against an inactive collateral.

However there exist a case in which GRAI can be minted against inactive collateral. As the `addCollateral`, `withdrawDebtTokens` and `adjustVessel` functions do not contain the `getIsActive` check, if a vessel is already opened then more GRAI tokens can be minted even after the asset gets set as inactive.

This can further be exploited by frontrunning the `setActive` call and opening a small amount vessel and then once the asset gets set as inactive, the attacker can borrow more GRAI against that inactive collateral.

## Attack scenario

Scenario 1:

- AssetA gets added as collateral in Gravita protocol.
- A user opens a vessel using AssetA as collateral.
- Protocol owner decides to disable AssetA as collateral (due to any possible reason like AssetA getting hacked or oracle failure)
- The user can still add more AssetA collateral to his vessel and borrow more GRAI against the newly deposited collateral.


Scenario 2:

- AssetA gets added as collateral in Gravita protocol.
- After a while the protocol owner decides to disable AssetA as collateral and submits the `setActive` transaction.
- An attacker sees owner's transaction and frontruns it to create a vessel using AssetA as collateral.
- So first the attackers `openVessel` gets executed followed by owner's `setActive`.
- Now the attacker can anytime add more AssetA collateral to his vessel and borrow more GRAI against the newly deposited collateral.

## Recommendation
Consider checking the `getIsActive` status of collateral in the `_adjustVessel` function.

