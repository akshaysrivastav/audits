# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/518
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L109
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L85-L86
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L105
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L92


# Vulnerability details

## Impact
The FixedPrice, LPDA & OpenEdition contracts uses `payable(address).transfer` method to send ETH which is unsafe.

[EIP1884](https://eips.ethereum.org/EIPS/eip-1884) increases the gas cost certain opcodes, possibly making contracts go over the 2300 gas limit by `transfer`, making them unable to receive funds via `transfer`. This can make the sale contracts unusable in the future. It is advised to use `address.call` instead.

## Proof of Concept
More details about the issue can be found at https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider using `address.call` to transfer ETH or use OpenZeppelin's Address library.