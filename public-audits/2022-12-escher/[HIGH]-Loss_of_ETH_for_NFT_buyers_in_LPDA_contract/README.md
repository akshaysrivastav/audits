# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/510
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L81-L88


# Vulnerability details

## Impact
The `buy` function of LPDA sale contract can be invoked with `0` as the input value and `0` ETH as the sent value(`msg.value = 0`). The `buy` function automatically ends the sale when `newId == sale.finalId` and distributes ETH to `feeReceiver` and `saleReceiver`.

Since the `buy` function do not validate the input `_amount` value, the contract is exposed to a critical issue which causes loss of ETH for the NFT buyer.

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

After the sale legitimately ends, the `buy` function can be invoked again with 0 as the input. This will re-execute the sale ending conditions of the LPDA sale contract as the `if` statement at `Line81` will result as `true`. The statements from `Line82-87` will execute again resulting in extra ETH transfer to the `feeReceiver` and `saleReceiver` addresses and an invalid `End` event emission. The extra ETH gets transferred from the refundable ETH balances of NFT buyers which they should ideally be able to claim using the `refund` function. But since the ETH are already being fractionally sent to `feeReceiver` and `saleReceiver`, the `refund` transactions of NFT buyers will start getting reverted. The `buy` function can be repeatedly invoked with 0 as input to drain the contract.

## Proof of Concept
Scenario:

 - Malicious attacker deploys a LPDA contract with params
      - saleReceiver: his own address
      - currentId: 0 
      - finalId: 2
      - sale duration: 10 seconds
      - startPrice: 10 ETH
      - dropPerSecond: 1 ETH
 - Alice buys first NFT just after the sale starts (at first second) at price of 10 ETH.
 - Alice buys second NFT just before the sale ends (at last second) at price of 1 ETH.
 - Sale should be over now with 1 ETH as the final price. Alice should now have refundable 10 ETH balance.
 - But now attacker invokes the `buy` function with 0 as input.
 - This causes the sale ending conditions to execute again, hence an additional 1.9 ETH gets transferred to the attacker (saleReceiver).
 - The `buy` function can be invoked again to drain the LPDA contract of ETH as much as possible.
 - Now when Alice tries to claim her refundable 8 ETH, the transaction gets reverted as the contract does not possess sufficient ETH to refund Alice.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider adding check for zero input. Also consider adding a state variable to record if the sale has ended and check its value before ending the sale.