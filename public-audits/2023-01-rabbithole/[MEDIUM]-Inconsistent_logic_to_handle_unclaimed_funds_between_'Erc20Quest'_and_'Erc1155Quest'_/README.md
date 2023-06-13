# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/178
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81-L87
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L54-L63


# Vulnerability details

## Impact
The `Erc20Quest` and `Erc1155Quest` contracts both inherit a common `Quest` contract. However the mechanism to handle unclaimed rewards is totally different in both the contracts.

Erc20Quest
```solidity
    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);
        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
    }
```
Erc1155Quest
```solidity
    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);
        IERC1155(rewardToken).safeTransferFrom(
            address(this),
            to_,
            rewardAmountInWeiOrTokenId,
            IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
            '0x00'
        );
    }
```

In Erc20Quest the unclaimed rewards of users are intended to stay in the contract so that the users can claim them anytime in the future. While in Erc1155Quest the `owner` can pull out all unclaimed rewards as soon as the Quest ends.

This raises two major concerns:
1. The behaviour of Erc20Quest and Erc1155Quest contracts (even though both inherits a common Quest contract) are totally different to each other w.r.t handling the unclaimed rewards.
2. For Erc1155Quest the owner can pull out all unclaimed rewards of the users. Resulting in loss for the users in case they ever come back to claim their rewards.

In lack of sufficient documentation both the two scenarios cannot be considered as intentional and should be treated as unintended results.

## Proof of Concept
Test cases were added to a new test file and ran using `npx hardhat test path-to-file`.

```typescript
import { expect } from 'chai'
import { ethers, upgrades } from 'hardhat'

const AddressZero = ethers.constants.AddressZero;

describe('Quest Bug', () => {
    it('Erc20Quest: User can claim unclaimed rewards in future', async () => {
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
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(240)                // balance of Quest = 240 tokens
        await ethers.provider.send('evm_increaseTime', [expiryDate])

        // Owner withdraws the extra tokens
        await QuestContract.withdrawRemainingTokens(owner.address);                             // 120 tokens were transferred to owner
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(120)

        // Owner withdraws protocol fee
        await QuestContract.withdrawFee();                                                      // 20 tokens were transferred to owner
        expect(await SampleERC20.balanceOf(owner.address)).to.equal(140)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(100)

        // User comes and claims pending unclaimed rewards successfully
        expect(await SampleERC20.balanceOf(user.address)).to.equal(0)                           // balance of user = 0 tokens
        await QuestContract.connect(user).claim();
        expect(await SampleERC20.balanceOf(user.address)).to.equal(100)                         // balance of user = 100 tokens
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(0)                  // balance of Quest = 0 tokens

        await ethers.provider.send('evm_increaseTime', [-startDate])
        await ethers.provider.send('evm_increaseTime', [-expiryDate])
    })

    it('Erc1155Quest: User cannot claim unclaimed rewards as owner pulled out all rewards', async () => {
        // Minimal Setup
        const [owner, user,] = await ethers.getSigners();
        const questA = 'questA'
        const erc1155Id = 1
        const expiryDate = Math.floor(Date.now() / 1000) + 100000
        const startDate = Math.floor(Date.now() / 1000) + 1000
        const RabbitHoleReceipt = (await upgrades.deployProxy((await ethers.getContractFactory('RabbitHoleReceipt')), [
            AddressZero, owner.address, AddressZero, 0
        ]))
        const QuestFactory = (await upgrades.deployProxy((await ethers.getContractFactory('QuestFactory')), [
            owner.address, RabbitHoleReceipt.address, owner.address
        ]))
        const SampleERC1155 = await (await ethers.getContractFactory('SampleErc1155')).deploy()

        // New Quest
        const createQuestTx = await QuestFactory.createQuest(
            SampleERC1155.address,
            expiryDate,
            startDate,
            2,          // max participant
            erc1155Id,  // tokenId
            'erc1155',
            questA
        );
        const waitedTx = await createQuestTx.wait();
        const event = waitedTx?.events?.find((event: any) => event.event === 'QuestCreated');
        const [, contractAddress] = event.args;
        const QuestContract = await ethers.getContractAt('Erc20Quest', contractAddress);

        // transferring required tokens to Quest contract
        await SampleERC1155.safeTransferFrom(owner.address, QuestContract.address, erc1155Id, 2, '0x')
        expect(await SampleERC1155.balanceOf(QuestContract.address, erc1155Id)).to.eq(2);
        await QuestContract.start();
        await ethers.provider.send('evm_increaseTime', [startDate])

        // Get RabbitHoleReceipt token
        const messageHash1 = ethers.utils.solidityKeccak256(['address', 'string'], [user.address.toLowerCase(), questA])
        const signature1 = await owner.signMessage(ethers.utils.arrayify(messageHash1))
        await QuestFactory.connect(user).mintReceipt(questA, messageHash1, signature1)
        expect(await RabbitHoleReceipt.balanceOf(user.address)).to.equal(1)

        // User didn't come to claim the rewards
        expect(await SampleERC1155.balanceOf(QuestContract.address, erc1155Id)).to.eq(2);           // balance of Quest = 2 tokens
        await ethers.provider.send('evm_increaseTime', [expiryDate])

        // Owner withdraws the extra tokens
        await QuestContract.withdrawRemainingTokens(owner.address);                                 // All tokens were transferred to owner
        expect(await SampleERC1155.balanceOf(QuestContract.address, erc1155Id)).to.equal(0)


        // User comes and tries to claim pending unclaimed rewards
        expect(await SampleERC1155.balanceOf(user.address, erc1155Id)).to.equal(0)                  // balance of user = 0 tokens
        await expect(QuestContract.connect(user).claim()).to.be
            .revertedWith('ERC1155: insufficient balance for transfer');
        expect(await SampleERC1155.balanceOf(user.address, erc1155Id)).to.equal(0)                  // balance of user = 0 tokens
        expect(await SampleERC1155.balanceOf(QuestContract.address, erc1155Id)).to.equal(0)         // balance of Quest = 0 tokens

        await ethers.provider.send('evm_increaseTime', [-startDate])
        await ethers.provider.send('evm_increaseTime', [-expiryDate])
    })
})
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
The logic for handling unclaimed rewards should be revisited and be made consistent in both the `Erc20Quest` and `Erc1155Quest` contracts. Also owner should not be allowed to pull out unclaimed user rewards in `Erc1155Quest` contract.
