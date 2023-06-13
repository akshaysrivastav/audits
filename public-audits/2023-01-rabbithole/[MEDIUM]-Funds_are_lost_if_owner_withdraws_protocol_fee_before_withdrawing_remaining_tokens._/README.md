# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/115
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81


# Vulnerability details

## Impact
The `Erc20Quest` contract has two functions by which `owner` should legitimately pull out funds from the contract.
1. `withdrawRemainingTokens` - to pull out any excess funds exlcuding protocol fee and unclaimed rewards.
2. `withdrawFee` - to pull out the protocol fee

```solidity
    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
    }
```

The `withdrawRemainingTokens` function calculates the output tokens by substracting the protocol fee and unclaimed tokens from the current token balance of the contract. This calculation works fine if the `withdrawRemainingTokens` function is invoked before the `withdrawFee` function.

However, if the `withdrawFee` function is invoked before the `withdrawRemainingTokens` function, the output amount is calculated incorrectly. As the the `withdrawRemainingTokens` function always substract the `protocolFee()` value (even when the protocol fee is already claimed) the output amount comes out to be less than the correct amount.

#### Actual Impact
If the `withdrawFee` function is invoked before the `withdrawRemainingTokens` function, then the token output amount becomes less than the correct amount and the difference between those amounts gets stuck in the contract forever (assuming both functions can be invoked only once). 

## Proof of Concept
A new test file was created and was ran using `npx hardhat test path-to-file`.
```typescript
import { expect } from 'chai'
import { ethers, upgrades } from 'hardhat'

const AddressZero = ethers.constants.AddressZero;

describe('Quest Bug', () => {
    it('loss of funds if protocol fee is withdrawn first', async () => {
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

        // 1 out of 2 user claims
        await QuestContract.connect(user).claim();
        expect(await SampleERC20.balanceOf(user.address)).to.equal(100)             // 100 tokens were transferred to user
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(140)    // balance of Quest = 240 - 100 = 140 tokens

        // withdraw protocol fee
        await QuestContract.withdrawFee();                                          // 20 tokens were transferred to owner
        expect(await SampleERC20.balanceOf(owner.address)).to.equal(20)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(120)

        // withdraw remaining tokens
        await QuestContract.withdrawRemainingTokens(owner.address);                 // 100 tokens were transferred to owner
        expect(await SampleERC20.balanceOf(owner.address)).to.equal(120)
        expect(await SampleERC20.balanceOf(QuestContract.address)).to.equal(20)     // 20 tokens are still present in Quest
    })
})
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
The contract should validate that remaining tokens must be distributed before distributing the protocol fee. 