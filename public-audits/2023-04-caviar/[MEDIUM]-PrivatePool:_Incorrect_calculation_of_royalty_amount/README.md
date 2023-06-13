# Original link
https://github.com/code-423n4/2023-04-caviar-findings/issues/580
# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L236-L249
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L274
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L335-L341


# Vulnerability details

## Impact
The `buy` and `sell` functions of PrivatePool intends to pay royalty amounts to royalty recipients on every token swap. The calculation of royalty amount is done as follows:

```solidity
    function buy(...)
        public
        payable
        returns (uint256 netInputAmount, uint256 feeAmount, uint256 protocolFeeAmount)
    {
        // ...

        // calculate the sale price (assume it's the same for each NFT even if weights differ)
        uint256 salePrice = (netInputAmount - feeAmount - protocolFeeAmount) / tokenIds.length;
        uint256 royaltyFeeAmount = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);
            if (payRoyalties) {
                (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], salePrice);
                royaltyFeeAmount += royaltyFee;
            }
        }
        
        // ...
        
        if (payRoyalties) {
            for (uint256 i = 0; i < tokenIds.length; i++) {
                (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);

                if (royaltyFee > 0 && recipient != address(0)) {
                    if (baseToken != address(0)) {
                        ERC20(baseToken).safeTransfer(recipient, royaltyFee);
                    } else {
                        recipient.safeTransferETH(royaltyFee);
                    }
                }
            }
        }
        // ...
    }
```

It can be seen that the a common `salePrice` value is taken for all NFTs. The comment also mentions that
> calculate the sale price (assume it's the same for each NFT even if weights differ)

This assumption is wrong and results in incorrect royalty amounts sent to royalty recipient. It should never be assumed that the royalty percentage of all NFTs will be equal. The profit/loss of royalty recipient is counter balanced with the funds of the trader. 

A detailed example is provided in the PoC section.

The bug results in direct loss of funds to the users interacting with the `buy` and `sell` functions, or the royalty recipient. As the calculations are similar in both `buy` and `sell` functions, both of them contain this bug.

## Proof of Concept
Consider the scenario in which two NFts (NFT1 and NFT2) are present in the pool.
Assumptions:
- The weights ratio of the two NFTs is 5:1
- For simplicity assume the worth of NFT1 is 10 ETH and NFT2 is 2 ETH
- Royalty fee of NFT1 is 10% and NFT2 is 20%.
- All other fee are assumed to be 0

Now suppose a `buy` transaction for these two NFTs happens in the pool.

In the ideal case, the royalty amounts for each NFT should be:
- NFT1 - 10% of 10 ETH = 1 ETH
- NFT2 - 20% of 2 ETH = 0.4 ETH
So a total of 1.4 ETH gets transferred to the royalty recipient.

But this is what happens with the current implementation of PrivatePool. 
- The `salePrice` is calculated as: (10 ETH + 2 ETH) / 2 = 6 ETH
- The royalty amounts comes out to be:
- NFT1 - 10% of 6 ETH = 0.6 ETH
- NFT2 - 20% of 6 ETH = 1.2 ETH
So a total of 1.8 ETH gets transferred to the royalty recipient (0.4 ETH more than the ideal amount).

This royalty amount including the incorrect & extra 0.4 ETH is collected from the buyer of the NFTs. Hence this results in loss of funds to the buyer. This lost amount gets transferred to the royalty recipient.

If the royalty percentages of the two NFTs are switched, then that will result in profit to trader and loss to royalty recipient.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider calculating the royalty amounts with respect to the individual price of each NFTs like `_getRoyalty(tokenIds[i], salePrice of tokenIds[i])`.