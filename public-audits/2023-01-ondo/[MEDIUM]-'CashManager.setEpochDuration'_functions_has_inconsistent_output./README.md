# Original link
https://github.com/code-423n4/2023-01-ondo-findings/issues/205
# Lines of code

https://github.com/code-423n4/2023-01-ondo/blob/main/contracts/cash/CashManager.sol#L546-L552


# Vulnerability details

## Impact
The CashManager contract contains `setEpochDuration` function which is used by `MANAGER_ADMIN` role to update the `epochDuration` parameter.

```solidity
  function setEpochDuration(uint256 _epochDuration) external onlyRole(MANAGER_ADMIN) {
    uint256 oldEpochDuration = epochDuration;
    epochDuration = _epochDuration;
    emit EpochDurationSet(oldEpochDuration, _epochDuration);
  }
```

The result of the `setEpochDuration` function execution can be impacted any external agent. The `epochDuration` is a crucial parameter of CashManager contract which determines the length of epochs in the contract.

The issue here is that the `setEpochDuration` function updates the `epochDuration` value without invoking the `transitionEpoch` function first. This leads to two different end results and scenarios:

1. When `transitionEpoch` is executed before `setEpochDuration` by an external agent (front-running).
2. When `transitionEpoch` is executed after `setEpochDuration` by an external agent (back-running).

In these two different cases, the duration and epoch number of last few passed epochs can be impacted differently. The result become dependent upon the wish of the external agent.

The exact impact is demonstrated in the PoC below.

## Proof of Concept
New test cases were added to `forge-tests/cash/cash_manager/Setters.t.sol` file.
```solidity
  function test_bug_inconsistentOutputOf_setEpochDuration_Case1() public {
    // skip 1 epoch duration
    vm.warp(block.timestamp + 1 days);

    // here the setEpochDuration() txn is frontrunned by issuing the transitionEpoch() txn
    cashManager.transitionEpoch();
    // this is the setEpochDuration() txn which was frontrunned
    cashManager.setEpochDuration(2 days);
    assertEq(cashManager.currentEpoch(), 1);

    vm.warp(block.timestamp + 2 days);
    cashManager.transitionEpoch();
    assertEq(cashManager.currentEpoch(), 2);   // number of epochs after 3 days is 2
  }
  function test_bug_inconsistentOutputOf_setEpochDuration_Case2() public {
    // skip 1 epoch duration
    vm.warp(block.timestamp + 1 days);
    
    // here we wait for the setEpochDuration() to be validated on the network
    cashManager.setEpochDuration(2 days);
    // then we backrun the setEpochDuration() txn with transitionEpoch() txn
    cashManager.transitionEpoch();
    assertEq(cashManager.currentEpoch(), 0);

    vm.warp(block.timestamp + 2 days);
    cashManager.transitionEpoch();
    assertEq(cashManager.currentEpoch(), 1);   // number of epochs after 3 days is 1
  }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
The `transitionEpoch` function should be executed before executing the `setEpochDuration` function so that the values for passed epochs are recorded in a consistent way. This can be done by adding the `updateEpoch` modifier.
```solidity
  function setEpochDuration(uint256 _epochDuration) external updateEpoch onlyRole(MANAGER_ADMIN) {
    uint256 oldEpochDuration = epochDuration;
    epochDuration = _epochDuration;
    emit EpochDurationSet(oldEpochDuration, _epochDuration);
  }
```