# Original link
https://github.com/code-423n4/2022-09-vtvl-findings/issues/279
1. `AccessProtected.setAdmin()` function should be marked as `external`.

2. `VariableSupplyERC20Token.mint()` - line 41 can be removed, solidity will automatically revert on underflow in the next line.
    ```javascript
    41    require(amount <= mintableSupply, "INVALID_AMOUNT");
    42    // We need to reduce the amount only if we're using the limit, if not just leave it be
    43    mintableSupply -= amount;
    ```

3. `VariableSupplyERC20Token.mint()` - line 37 can be removed, `ERC20._mint()` already checks the zero address condition.
    ```javascript
    37    require(account != address(0), "INVALID_ADDRESS");
    ```

4. VTVLVesting - no need to explicitly initialize variable with 0 value, default value for all uints in Solidity is 0. This issue is present at Line 27 and Line 148.
    ```
    27    uint112 public numTokensReservedForVesting = 0;
    ```
    ```
    148    uint112 vestAmt = 0;
    ```

5. In `VTVLVesting.createClaimsBatch()`, it will be better to use `Claim[]` array as input parameter. This way [explicit length checking](https://github.com/code-423n4/2022-09-vtvl/blob/f68b7f3e61dad0d873b5b5a1e8126b839afeab5f/contracts/VTVLVesting.sol#L344-L351) won't be needed.

6. `VTVLVesting.withdrawAdmin()` - no need for line 402. 

    `tokenAddress.safeTransfer()` will automatically revert on insufficient balance.
    ```javascript
    402    require(amountRemaining >= _amountRequested, "INSUFFICIENT_BALANCE");
    ```

7. `VTVLVesting.withdrawOtherToken()` - no need for line 449.

    `_otherTokenAddress.safeTransfer()` will automatically revert on insufficient balance.
    ```javascript
    449    require(bal > 0, "INSUFFICIENT_BALANCE");
    ```

8. The `hasNoClaim` modifier in VTVLVesting is only used once so it is better to use the [`require`](https://github.com/code-423n4/2022-09-vtvl/blob/f68b7f3e61dad0d873b5b5a1e8126b839afeab5f/contracts/VTVLVesting.sol#L129) statement itself in `_createClaimUnchecked()` function and remove the modifier from the code.


