# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/328
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L73
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L81-L88


# Vulnerability details

## Impact
In the `buy` function of FixedPrice and LPDA contracts the transfer of funds to `saleReceiver` only happens when `newId` becomes equal to `finalId`, i.e, when the entire batch of NFTs gets sold completely.

FixedPrice :
```solidity
        if (newId == sale_.finalId) _end(sale);
```
LPDA :
```solidity
        if (newId == temp.finalId) {
            sale.finalPrice = uint80(price);
            uint256 totalSale = price * amountSold;
            uint256 fee = totalSale / 20;
            ISaleFactory(factory).feeReceiver().transfer(fee);
            temp.saleReceiver.transfer(totalSale - fee);
            _end();
        }
```
This logic can cause issues if only a subset of NFTs get sold. In that case the ETH collected by contract from the sales of NFTs will get stuck inside the contract. 

There is no function inside those contracts to recover the ETH collected from the partial sale of the NFT batch. There is also no function to send back those ETH to the NFT buyers. This causes the ETH to be in a stuck state as they cannot be pulled from the contracts. 

The only way to recover those ETH is to explicitly buy the entire batch of to-be-sold NFTs so that the contract automatically triggers the sale ending conditions. This recovery method may require a decent amount of upfront capital to buy all unsold NFTs.


## Proof of Concept
Consider this scenario:

 - The sale contract deployer deploys a sale contract (FixedSale) to sell 100 NFTs ranging from id 1 to 100 with price 1 ETH per NFT.
 - Users only buy the initial 10 NFTs. So the sale contract now has 10 ETH and currently 90 NFTs are unsold. 
 - No more users buy anymore NFTs from the sale contract.
 - The 10 ETH are in stuck state as the `saleReceiver` cannot pull them out even so some NFTs have been already legitimately sold.
 - To recover those 10 ETH, the contract deployer or `saleReceiver` has to come up with 90 ETH and buy the unsold 90 NFTs.

Similar is the scenario for LPDA sale contract. ETH can get stuck for partial batch NFT sales and upfront capital will be needed to recover the ETH.  

## Tools Used
Manual review

## Recommended Mitigation Steps
The contracts can have an additional function to claim the ETH collected from already successful NFT sales.
```solidity
    function claim() external onlyOwner {
        ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);
        sale.saleReceiver.transfer(address(this).balance);
    }
```
Please note, use of `transfer` is not recommended. The above code is just an example mitigation.