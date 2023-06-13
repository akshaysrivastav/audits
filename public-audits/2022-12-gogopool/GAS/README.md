# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/582
1. In ERC20Upgradeable contract `INITIAL_CHAIN_ID` and `INITIAL_DOMAIN_SEPARATOR` can be made `immutable`.
    ```solidity
    	uint256 internal INITIAL_CHAIN_ID;
	    bytes32 internal INITIAL_DOMAIN_SEPARATOR;
	```
2. In Storage.sol at L63 `msg.sender` should be used instead of reading `newGuardian` from storage.
    ```solidity
    	if (msg.sender != newGuardian) {
			revert InvalidGuardianConfirmation();
		}
		// ...
		guardian = newGuardian;
    ```

3. MultisigManager - at L74 `deleteBool` should be used as the `delete` keyword uses less gas.
    ```solidity
    74		setBool(keccak256(abi.encodePacked("multisig.item", multisigIndex, ".enabled")), false);
    ```
    
4. ProtocolDAO - at L74 `deleteBool` should be used as the `delete` keyword uses less gas.
    ```solidity
    74		setBool(keccak256(abi.encodePacked("contract.paused", contractName)), false);
    ```
    
5. In `MultisigManager.requireNextActiveMultisig` L82 can be omitted and `addr` can be declared in return datatypes.
    ```solidity
	function requireNextActiveMultisig() external view returns (address) {
		uint256 total = getUint(keccak256("multisig.count"));
		address addr;       // L82
		bool enabled;
		for (uint256 i = 0; i < total; i++) {
			(addr, enabled) = getMultisig(i);
			if (enabled) {
				return addr;
			}
		}
		revert NoEnabledMultisigFound();
	}
    ```
6. In `Vault.withdrawAVAX` the second `if` condition (L71-L73) can be omitted. As the condition is already validated by solidity's underflow protection. Same is the case in `transferAVAX`, `withdrawToken`, `transferToken` functions.
7. In `Vault.removeAllowedToken` `delete` should be used as the `delete` keyword uses less gas.
    ```solidity
    209		allowedTokens[tokenAddress] = false;
    ```
8. In `Vault.withdrawToken` at L157 there is no need to create new `tokenContract` variable in memory as `tokenAddress` is already of type `ERC20`.
    ```solidity
    157		ERC20 tokenContract = ERC20(tokenAddress);
    ```
9. In Vault contract `onlyRegisteredNetworkContract` modifier can be removed from `withdraw` functions as the deposit balance of caller is validated.
10. In Staking contract `increaseGGPStake` function can be omitted and the needed statements can be written directly in the `_stakeGGP` function. The `increaseGGPStake` is only called once.
    ```solidity
    function increaseGGPStake(address stakerAddr, uint256 amount) internal {
		int256 stakerIndex = requireValidStaker(stakerAddr);
		addUint(keccak256(abi.encodePacked("staker.item", stakerIndex, ".ggpStaked")), amount);
	}
    ```
11. MinipoolManager - In `onlyOwner` modifier `msg.sender` should be returned instead of memory variable.
    ```solidity
    function onlyOwner(int256 minipoolIndex) private view returns (address) {
		address owner = getAddress(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".owner")));
		if (msg.sender != owner) {
			revert OnlyOwner();
		}
		return owner;
	}
    ```
12. MinipoolManager - In `requireValidStateTransition` function there is no need for the last `else` block.
    ```solidity
    function requireValidStateTransition(int256 minipoolIndex, MinipoolStatus to) private view {
		// ...
		bool isValid;
		if (currentStatus == MinipoolStatus.Prelaunch) {
			isValid = (to == MinipoolStatus.Launched || to == MinipoolStatus.Canceled);
		} else if (currentStatus == MinipoolStatus.Launched) {
		} else if {...}
		else {
			isValid = false;
		}
        // ...
	}
    ```
13. Since `Vault` and `TokenGGP`  contracts are non-upgradeable contracts, their addresses can be stored in `immutable` variables in other contracts instead of reading those addresses from the `Storage` contract.
    ```solidity
    Vault vault = Vault(getContractAddress("Vault"));
    ```
    ```solidity
    TokenGGP ggpToken = TokenGGP(getContractAddress("TokenGGP"));
    ```
14. In `ClaimNodeOp.calculateAndDistributeRewards` the `rewardsPool.getRewardsCycleCount()` value should be cached.
    ```solidity
    	if (rewardsPool.getRewardsCycleCount() == 0) {
			revert RewardsCycleNotStarted();
		}

		if (staking.getLastRewardsCycleCompleted(stakerAddr) == rewardsPool.getRewardsCycleCount()) {
			revert RewardsAlreadyDistributedToStaker(stakerAddr);
		}
		staking.setLastRewardsCycleCompleted(stakerAddr, rewardsPool.getRewardsCycleCount());
    ```
15. In `TokenggAVAX.depositFromStaking` the `msg.value` should be used instead of `totalAmt`.
    ```solidity
    function depositFromStaking(uint256 baseAmt, uint256 rewardAmt) public ... {
		uint256 totalAmt = msg.value;
		if (totalAmt != (baseAmt + rewardAmt) || baseAmt > stakingTotalAssets) {
			revert InvalidStakingDeposit();
		}
        // ...
		IWAVAX(address(asset)).deposit{value: totalAmt}();
	}
    ```
