# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/350
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L63


# Vulnerability details

## Impact
The `buy` function of `OpenEdition` declares variables which are shorter than 32 bytes and it also performs arithmetic operations on those variables. The function declares an `uint24 amount` variable and performs its multiplication with an `uint72 sale.price` variable.

```solidity
    function buy(uint256 _amount) external payable {
        uint24 amount = uint24(_amount);
       
        // ...

        require(amount * sale.price == msg.value, "WRONG PRICE");

        // ...
    }
```
The max value which the multiplication operation can store is `type(uint72).max` which is `4,722,366,482,869,645,213,695`. This max limit can be easily hit by calculations involving ETH values which are in 18 decimals. With respect to ETH the number is close to 4722.36 ETH. Any buy call which transfers more than this value of ETH will result in transaction getting reverted. 

## Proof of Concept
Scenario:
 - An `OpenEdition` contract is deployed to sell 10000 NFTs each priced at 5 ETH.
 - A user tries to buy 1000 NFTs.
 - The multiplication result of this operation, 1000 * 5e18, exceeds the max limit of `uint72`.
 - Hence the transaction gets reverted.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider increasing the size of `uints` for the `amount` and `price` variables.