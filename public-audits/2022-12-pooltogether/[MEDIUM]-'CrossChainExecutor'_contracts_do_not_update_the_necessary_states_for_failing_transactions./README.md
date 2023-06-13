# Original link
https://github.com/code-423n4/2022-12-pooltogether-findings/issues/166
# Lines of code

https://github.com/pooltogether/ERC5164/blob/5647bd84f2a6d1a37f41394874d567e45a97bf48/src/ethereum-optimism/EthereumToOptimismExecutor.sol#L45-L59
https://github.com/pooltogether/ERC5164/blob/5647bd84f2a6d1a37f41394874d567e45a97bf48/src/ethereum-arbitrum/EthereumToArbitrumExecutor.sol#L31-L45


# Vulnerability details

## Impact
The `CrossChainExecutorArbitrum` and `CrossChainExecutorOptimism` contracts both use `CallLib` library to invoke `Call`s on external contract. As per the `CallLib` library implementation, any failing `Call` results in the entire transaction getting reverted.

The `CrossChainExecutor` contracts does not store whether the calls in `CallLib.Call[]` were already attempted which failed. 

This creates several issues for `CrossChainExecutor` contracts.

 1. Offchain components can be tricked to submit failing `Call[]`s again and again. This can be used to drain the offchain component of gas. 

 2. Once a failing `Call[]` was invoked (which failed) and if again the same `Call[]` is invoked, the transaction should revert with `CallsAlreadyExecuted` error but it reverts with `CallFailure` error.

3. It is difficult to determine whether a to-be executed `Call[]` is pending or the invocation was already tried but failed. 

PoCs for the above issues are listed below.


## Proof of Concept

#### Scenario 1
```solidity
contract Foo {
    function bar() public {
        for(uint256 i; ; i++) {}
    }
}
```
 - The attacker relays the `Foo.bar()` call in the `CrossChainRelayer` contract with `maxGasLimit` as the `_gasLimit` parameter.
 - The transport layer tries to invoke the `Foo.bar()` call by calling the `CrossChainExecutor.executeCalls()`. This transaction reverts costing the transport layer client `maxGasLimit` gas.
 - Since no state updates were performed in `CrossChainExecutor`, the transport layer still assumes the relayed call as pending which needs to be executed. The transport layer client again tries to execute the pending relayed call which reverts again.
- Repeated execution of the above steps can deplete the gas reserves of transport layer client.

#### Scenario 2
```solidity
contract Foo {
    function bar() public {
        revert();
    }
}
```
 - The attacker relays the `Foo.bar()` call in the `CrossChainRelayer` contract.
 - The transport layer tries to invoke the `Foo.bar()` call by calling the `CrossChainExecutor.executeCalls()`. This transaction gets reverted.
 - Since the relayed calls still seems as pending, the transport layer tries to invoke the `Foo.bar()` call again. This call should get reverted with `CallsAlreadyExecuted` error but it gets reverted with `CallFailure` error.

## Tools Used
Manual review

## Recommended Mitigation Steps
The `CrossChainExecutor` contract should store whether a relayed call was attempted to be executed to make sure the execution cannot be tried again.

The `CallLib` library can be changed to not completely revert the transaction when any individual `Call` gets failed. 