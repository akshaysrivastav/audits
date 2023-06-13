# Original link
https://github.com/code-423n4/2023-04-caviar-findings/issues/596
# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L237-L281
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L328-L355


# Vulnerability details

## Impact
The `PrivatePool.buy` and `PrivatePool.sell` functions intend to distribute royalty amount whenever NFTs are traded. The implementation of buy and sell looks like this:
```solidity
    function buy(uint256[] calldata tokenIds, uint256[] calldata tokenWeights, MerkleMultiProof calldata proof)
        public
        payable
        returns (uint256 netInputAmount, uint256 feeAmount, uint256 protocolFeeAmount)
    {
        // ...

        // calculate the sale price (assume it's the same for each NFT even if weights differ)
        uint256 salePrice = (netInputAmount - feeAmount - protocolFeeAmount) / tokenIds.length;
        uint256 royaltyFeeAmount = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            // transfer the NFT to the caller
            ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);

            if (payRoyalties) {
                // get the royalty fee for the NFT
                (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], salePrice);

                // add the royalty fee to the total royalty fee amount
                royaltyFeeAmount += royaltyFee;
            }
        }

        // add the royalty fee amount to the net input aount
        netInputAmount += royaltyFeeAmount;

        // ...

        if (payRoyalties) {
            for (uint256 i = 0; i < tokenIds.length; i++) {
                // get the royalty fee for the NFT
                (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);

                // transfer the royalty fee to the recipient if it's greater than 0
                if (royaltyFee > 0 && recipient != address(0)) {
                    if (baseToken != address(0)) {
                        ERC20(baseToken).safeTransfer(recipient, royaltyFee);
                    } else {
                        recipient.safeTransferETH(royaltyFee);
                    }
                }
            }
        }

        // emit the buy event
        emit Buy(tokenIds, tokenWeights, netInputAmount, feeAmount, protocolFeeAmount, royaltyFeeAmount);
    }

    function sell(
        ...
    ) public returns (...) {
        // ...

        uint256 royaltyFeeAmount = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            // transfer each nft from the caller
            ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);

            if (payRoyalties) {
                // calculate the sale price (assume it's the same for each NFT even if weights differ)
                uint256 salePrice = (netOutputAmount + feeAmount + protocolFeeAmount) / tokenIds.length;

                // get the royalty fee for the NFT
                (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);

                // tally the royalty fee amount
                royaltyFeeAmount += royaltyFee;

                // transfer the royalty fee to the recipient if it's greater than 0
                if (royaltyFee > 0 && recipient != address(0)) {
                    if (baseToken != address(0)) {
                        ERC20(baseToken).safeTransfer(recipient, royaltyFee);
                    } else {
                        recipient.safeTransferETH(royaltyFee);
                    }
                }
            }
        }

        // subtract the royalty fee amount from the net output amount
        netOutputAmount -= royaltyFeeAmount;

        if (baseToken == address(0)) {
            // transfer ETH to the caller
            msg.sender.safeTransferETH(netOutputAmount);

            // if the protocol fee is set then pay the protocol fee
            if (protocolFeeAmount > 0) factory.safeTransferETH(protocolFeeAmount);
        } else {
            // transfer base tokens to the caller
            ERC20(baseToken).transfer(msg.sender, netOutputAmount);

            // if the protocol fee is set then pay the protocol fee
            if (protocolFeeAmount > 0) ERC20(baseToken).safeTransfer(address(factory), protocolFeeAmount);
        }

        // ...
    }
```

It should be noted that while calculating `royaltyFeeAmount` the the `recipient` address returned from `_getRoyalty` function is ignored and the returned `royaltyFee` is added to the `royaltyFeeAmount`. This cumulative royalty amount is then collected from the trader.

However while performing the actual royalty transfer to the royalty recipient the returned `recipient` address is validated to not be equal to 0. The royalty is only paid when the `recipient` address is non-zero.

This inconsistency between royalty collection and royalty distribution can cause loss of funds to the traders. In the cases when `royaltyFee` is non-zero but `recipient` address is zero, the fee will be collected from traders but won't be distributed to royalty recipient. Hence causing loss of funds to the traders.
 
As the creation of private pools is open to everyone, the likelyhood of this vulnerability is high.

## Proof of Concept
Consider this scenario:
- A buyer initiates the `buy` call for an NFT.
- The `PrivatePool.buy` function queries the `_getRoyalty` function which returns 10 WETH as the `royaltyFee` and `0x00` address as the royalty recipient.
- This 10 WETH value will be added to the `royaltyFeeAmount` amount and will be collected from the buyer.
- But since the recipient address is `0x00`, the 10 WETH royalty amount will not be distributed. 
- The 10 WETH amount won't be returned to the buyer either. It just simply stays inside the pool contract.
- The buyer here suffered loss of 10 WETH.

A similar scenario is possible for the NFT `sell` flow.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider collecting royalty amount from traders only when the royalty recipient is non-zero.
```solidity
    if (royaltyFee > 0 && recipient != address(0)) {
        royaltyFeeAmount += royaltyFee;
    }
```
