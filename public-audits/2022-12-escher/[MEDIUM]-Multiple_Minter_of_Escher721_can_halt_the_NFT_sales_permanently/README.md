# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/335
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L65-L67
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L73-L75
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L66-L68


# Vulnerability details

## Impact
 As `Escher721` contract uses `AccessControlUpgradeable`, `MINTER_ROLE` can be given to multiple addresses. This multiple minter scenario can cause severe issues in the NFT Sale (ISale) contracts.
Suppose, an `Escher721` contract with multiple minters is created. A sale contract is deployed to sell NFTs from id 1 to 10. Any one of the existing multiple minters of `Escher721` externally mints the NFT of `id` 6 using the `Escher721.mint` function. As all the sale contracts (FixedSale, OpenEdition & LPDA) incrementally mints NFTs on every NFT sale, the sale contract will revert the transactions which tries to mint the NFT `id` 6 again. 
```solidity
        for (uint48 x = sale_.currentId + 1; x <= newId; x++) {
            nft.mint(msg.sender, x);
        }
``` 

Due to this, the sale contract will get stuck in a 'halt' state as it cannot mint and sell more NFTs. The ETH collected from already sold NFTs will also get stuck inside the contract.

## Proof of Concept
Scenario
 - Alice deploys `Escher721` contract and registers herself as a minter (`MINTER_ROLE`).
 - Alice also deploys a `FixedSale` contract to sell Escher721 NFTs from ids 0 to 99.
 - Since Alice has the `MINTER_ROLE` for the `Escher721` contract she mints id 25 to a account.
 - Now users come and buy NFTs upto id 24, the sale contract collects ETH payments for the sold NFTs.
 - A user comes and tries to buy more NFTs from the sale contract. His transaction will get reverted as NFT id 25 is already minted.
 - Now sale contract cannot mint and sell anymore NFTs. The already received ETH in the contract gets stuck with no way to recover them ever.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider giving the minting rights to a single Sale contract only so that any other account cannot mint NFTs externally.