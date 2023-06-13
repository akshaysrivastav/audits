# Original link
https://github.com/code-423n4/2022-12-pooltogether-findings/issues/177
 1. `nonce` increment statements can be modified to save gas.
    The statements in `relayCalls` functions of `CrossChainRelayerOptimism`, `CrossChainRelayerArbitrum` and `CrossChainRelayerPolygon` contracts:
    ```solidity
        nonce++;
    
        uint256 _nonce = nonce;
    ```
    can be replaced with
    ```solidity
        uint _nonce = ++nonce;
    ```

 2. `executor` variable should be made `immutable`.
 The `executor` variables in `CrossChainRelayerOptimism` and `CrossChainRelayerArbitrum` contracts can be made `immutable`. The address of the `executor` contract can be calculated deterministically before deployment.

 3.  `relayer` variable should be made `immutable`.
 The `relayer` variables in `CrossChainExecutorOptimism` and `CrossChainExecutorArbitrum` contracts can be made `immutable`. The address of the `relayer` contract can be calculated deterministically before deployment or it may be already be known before `CrossChainExecutor` deployments.