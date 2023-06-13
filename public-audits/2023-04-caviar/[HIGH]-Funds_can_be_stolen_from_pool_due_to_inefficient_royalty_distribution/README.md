# Original link
https://github.com/code-423n4/2023-04-caviar-findings/issues/593
# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L244
https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L274


# Vulnerability details

## Impact
The `PrivatePool.buy` function intends to distribute royalty amount whenever NFTs are bought. Its implementation looks like this:
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
    
    function _getRoyalty(uint256 tokenId, uint256 salePrice)
        internal
        view
        returns (uint256 royaltyFee, address recipient)
    {
        // get the royalty lookup address
        address lookupAddress = IRoyaltyRegistry(royaltyRegistry).getRoyaltyLookupAddress(nft);

        if (IERC2981(lookupAddress).supportsInterface(type(IERC2981).interfaceId)) {
            // get the royalty fee from the registry
            (recipient, royaltyFee) = IERC2981(lookupAddress).royaltyInfo(tokenId, salePrice);

            // revert if the royalty fee is greater than the sale price
            if (royaltyFee > salePrice) revert InvalidRoyaltyFee();
        }
    }
```

It should be noted that the collection of royalty amount from buyer [(L237 - L269)](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L237-L269) and distribution of royalty amounts to royalty recipient [(L271 - L285)](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L271-L285) happens separately. Also while distributing the royalty amount to recipient the royalty amount value is queried again.

This implementation can be misused to steal `baseToken` funds from the pool. In the case where the royalty amount returned in the second `_getRoyalty` call is greater than the amount return in the first `_getRoyalty` call, the sufficient royalty amount won't be collected from the buyer so the extra royalty amount needed will be sent from the PrivatePool's `baseToken` balance.

In the case where the royalty recipient has influence over the `royaltyRegistry`, `lookupAddress` or the `royaltyInfo` call, he can manipulate the result of second `_getRoyalty` call. If the result of second query is greater than the first one then the pool's own `baseToken` balance will transferred to the royalty recipient.

This bug will cause loss of funds to all the depositors of the pool as the originally deposited `baseToken`s and the pool's swap fee reserves can be stolen using this vulnerability. 

It should be noted that Caviar protocol intends to support open creation of private pools with any configuration possible as desired by the user. Hence the likelyhood of this exploit is high.

## Proof of Concept
Consider this scenario:
- Pool was created and 100 WETH were deposited by owner and other depositors.
- After some NFT swaps the pool accrued additional 10 WETH as the swap fee.
- A malicious user or someone with an influence over the `royaltyRegistry`, `lookupAddress` or the `royaltyInfo` call invokes the `buy` function.
- The `PrivatePool.buy` function queries and calculates the royalty amount, the `royaltyFeeAmount` comes out to be 10 WETH.
- These 10 WETH are collected from the buyer.
- The buy function makes the second call to the `_getRoyalty` function before the royalty amount transfer. But this time the amount returned comes out to be 20 WETH.
- The contract now proceeds with the royalty payment of 20 WETH out of which only 10 WETH were collected from the buyer and rest of 10 WETH are from pool's own 110 WETH balance.
- In this case the pool suffers a loss of 10 WETH.
- The attack cycle can be repeated again to steal more funds.

Note that the funds required to initiate the buy call and its recovery was omitted to make the scenario easier to understand. Any funds needed for the buy call can be recovered by performing further swaps.

## Tools Used
Manual review

## Recommended Mitigation Steps
Cosnider performing the royalty amount calculation, amount transfer from buyer to pool and amount transfer from pool to royalty recipient in a single `for` loop, just like how it is done in the `sell` function.
```solidity
    function sell(
        ...
    ) public returns (uint256 netOutputAmount, uint256 feeAmount, uint256 protocolFeeAmount) {
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

        // ...
    }
```
