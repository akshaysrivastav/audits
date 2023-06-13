# Original link
https://github.com/code-423n4/2023-02-ethos-findings/issues/447
# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L243-L251


# Vulnerability details

## Impact
The `ActivePool` contract deploys the idle collateral to yield generating vault. The implementation of the `_rebalance` function is such that it assumes that the Vault to which collateral was deployed never makes loss.

The code segment [L243 - L251](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L243-L251) of ActivePool.sol is as follows:
```solidity
    vars.currentAllocated = yieldingAmount[_collateral];
    
    // ...
    vars.ownedShares = vars.yieldGenerator.balanceOf(address(this));
    vars.sharesToAssets = vars.yieldGenerator.convertToAssets(vars.ownedShares);
    
    vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
```

The statement `vars.sharesToAssets.sub(vars.currentAllocated)` above states that the Vault is assumed to never make loss. This is an over-optimistic and bad assumption.

If due to any possible reason (like a hack) the Vault contract turns out to be lossy, the `_rebalance` function will start reverting due to the use of underflow protected `sub` function. In a lossy Vault's case the `sharesToAssets` value can become smaller than the `currentAllocated` value, due to this the `sub` will revert.

The `sendCollateral` and `pullCollateralFromBorrowerOperationsOrDefaultPool` functions of ActivePool (which internally call `_rebalance`) are used in most crucial operations of the Ethos protocol (like `openTrove`, `adjustTrove`, `closeTrove`, etc). A DoS of these function will make the Ethos protocol unusable.

The ReaperStrategyGranarySupplyOnly deploys collateral on Granary Finance which is a fork of Aave. In the past bugs have been found in the Aave contract which could have resulted in loss of funds. Considering that, the assumption that the Vault and ReaperStrategyGranarySupplyOnly can never make loss is totally unsafe.

## Proof of Concept
Consider this scenario:
 - Due to any technical hack, economical attack, MEV, market conditions, etc, the Vault suffers an economical loss.
 - Due to this the `sharesToAssets` value becomes smaller than `currentAllocated` value (even by a very small margin).
 - This will cause `_rebalance` to always revert. The `sendCollateral` and `pullCollateralFromBorrowerOperationsOrDefaultPool` functions will always revert. Essentially reverting all the crucial Ethos protocol operations like `openTrove`, `adjustTrove`, `closeTrove`, etc.
 - The protocol is in complete DoS state.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider handling the case when the Vault is lossy.
Additionally a fix like this can also be added:
```solidity
    if (vars.sharesToAssets < vars.currentAllocated) {
        vars.profit = 0;
    } else {
        vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
    }
```