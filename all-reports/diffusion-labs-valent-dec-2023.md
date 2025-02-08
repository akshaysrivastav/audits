# Introduction

A time-boxed security review of the **[Valent](https://diffusionlabs.xyz/)** protocol was done by **Akshay Srivastav**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort with a goal of finding as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with the smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Akshay Srivastav**

Akshay Srivastav is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@akshaysrivastv](https://twitter.com/akshaysrivastv).

# About **Valent**

It is a liquidation-free, oracle-less, immutable, permission-less, and fixed-term/fixed-rate lending/borrowing protocol.


# Security Assessment Summary

- **valent-monorepo - [d3f4a91fbe8ec4e3f479f63d67379fffae8ab55a](https://github.com/Diffusion-Labs/valent-monorepo/tree/d3f4a91fbe8ec4e3f479f63d67379fffae8ab55a)**

### Scope

The following smart contracts were in scope of the audit:

- All contracts in [packages/contracts/src](https://github.com/Diffusion-Labs/valent-monorepo/tree/d3f4a91fbe8ec4e3f479f63d67379fffae8ab55a/packages/contracts/src)


---


# Findings Summary

The following number of issues were found, categorized by their severity:

| Severity    | Issues    |
| ----------- | :-------: |
| High        | 2         |
| Medium      | 1         |
| Low         | 6         |
| Info        | 0         |


---

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [H-01] | Attacker can manipulate amms and sell user's entire collateral for profit   | High   |
| [H-02] | Force revert `Loan.repay` by repaying 1 wei loan amount   | High   |
| [M-01] | Loan can be repaid from any account by changing the loan's borrower   | Medium   |
| [L-01] | Uninitialized implementation contracts      | Low      |
| [L-02] | Lenders can update status and expiry of null collection      | Low      |
| [L-03] | Lack of input validation in `LoanExecutor.createLoans`      | Low      |
| [L-04] | Lack of input validation in `LoanExecutor.repayLoan`      | Low      |
| [L-05] | Use of default `0` value for `sqrtPriceLimitX96` parameter      | Low      |
| [L-06] | Revert on zero value token approvals      | Low      |


# Detailed Findings

## [H-01] Attacker can manipulate amms and sell user's entire collateral for profit.
### Severity
High

### Description
The `LoanExecutor.repayLoan` function let's users repay their loans by selling off their deposited collateral on Uniswap. Currently this is an open function which can be called by anyone.

As the function does not validate the caller, an attacker can call this function so that any user's collateral can be sold off to repay that user's debt. The attacker will generate profit by sandwiching the selling off of collateral.

All active loans can be repaid using this attack leading to losses for users.

#### Attack Scenario
- Suppose Alice has deposited $1000 worth of collateral and taken $500 worth of debt.
- Alice has also given collateral token allowance to the `LoanExecutor` contract.
- The attacker will call the `LoanExecutor.repayLoan` with Alice's loan details and a high `amountInMax` swapper parameter.
- Here is what will happen in the call:
    - Alice's wallet will receive $1000 collateral.
    - LoanExecutor will pull $1000 collateral from Alice wallet and sell it for $500 loan tokens.
    - Loan of $500 will be repaid.
- To take profit from this attack the attacker will manipulate the AMM pool so that the $1000 worth collateral can be sold for $500 worth value.
- The attacker can sandwich the `repayLoan` call to generate profit from the collateral sale.
- Alice suffers the loss of $500 worth of collateral.



### Recommendations
Consider validating the caller of `LoanExecutor.repayLoan` call, only the loan's borrower should be allowed to execute this function.

### Remark
Fixed. Now only the borrower can call `LoanExecutor.repayLoan`.

## [H-02] Force revert `Loan.repay` by repaying 1 wei loan amount.
### Severity
High

### Description
The `Loan.repay` function looks like this:

```solidity
  function repay(uint256 repayAmt, bytes calldata data) external nonReentrant {
    /// @dev check if loan is expired
    if (Helper.checkExpired(loanData.expiresAt)) revert Errors.LoanExpired();

    /// @dev check if not, is there any repayment amount left?
    uint256 remainingAmt = loanData.borrowAmt - loanData.repaidAmt;

    if (repayAmt == 0) revert Errors.ZeroAmtProvided();
    if (repayAmt > remainingAmt) revert Errors.AmtExceedsBalance();     // @audit bug here
    ...
  }
```

The highlighted statement above enforces that the repay amount cannot be greater than the remaining repay amount.

This condition can be misused to forcefully revert a loan repayment transaction.

As per the design of protocol, there is no advantage to borrowers for paying off their debts before the expiry. So it is likely that most borrowers will repay their debts near the expiry. Hence forcefully delaying the repayment can make the loan go past the expiry timestamp resulting in collateral seizure.

### Scenario
- Suppose a governance controlled entity took a loan of 100K USDC against their collateral.
- Near the end of loan duration, the entity proposes a loan repayment proposal to its governance.
- The governance takes a couple of days for voting to succeed.
- Now we are almost near the expiry timestamp and governance is ready to repay the full loan.
- A malicious user repays a debt amount of 1 wei for the entity.
- Now the full repayment txn (100K USDC) of governance entity gets reverted.
- There won't be enough time to propose a new proposal and get done with voting before the loan expires.
- Loan gets expired and collateral gets seized.


### Recommendations
In the repay function, if the `repayAmt` is more remaining payment then consider doing this:
```solidity
    if (repayAmt > remainingAmt) {
        repayAmt = remainingAmt
    }
``` 

### Remark
Fixed. The `previewRepay` function is now used which correctly handles the maximum repay amount scenario.


## [M-01] Loan can be repaid from any account by changing the loan's borrower.
### Severity
Medium

### Description
See these code segments:

Loan.sol
```solidity
  function transfer(address newBorrower) external {
    ...
    loanData.borrower = newBorrower;
  }

  function repay(uint256 repayAmt, bytes calldata data) external nonReentrant {
    ...
    IERC20Decimals(loanData.collToken).safeTransfer(loanData.borrower, reclaimedAmt);
    if (data.length != 0) {
      ILoanTaker(msg.sender).sourcePrincipal(repayAmt, reclaimedAmt, data);
    }
    IERC20Decimals(loanData.borrowToken).safeTransferFrom(msg.sender, address(loanData.lenderVault), repayAmt);
  }
```

LoanExecutor.sol
```solidity
  function sourcePrincipal(uint256 repayAmt, uint256 reclaimedColl, bytes calldata data) external {
    ...
    (address borrower,, address collToken, address borrowToken,,,,) = ILoan(lrp.loan).loanData();

    IERC20Decimals(collToken).safeTransferFrom(borrower, lrp.swapper, reclaimedColl);

    ISwapper(lrp.swapper).swapOut(collToken, borrowToken, repayAmt, lrp.swapperData);
  }
```

There are two things to note:
- In the `LoanExecutor.repayLoan` call:
    - Collateral tokens gets transferred to `loanData.borrower`.
    - In `LoanExecutor.sourcePrincipal` collateral tokens gets pulled from `loanData.borrower` for swapping.
- The `Loan` contract allows transferring of loans to a new borrower.

The combination of these two features can be used to repay loan from any account who has given token approvals to `LoanExecutor`. Note that this does not cause loss of funds to any party. But still having the ability to move idle funds from a user's wallet is a critical issue.

### Scenario
- Alice has deposited some collateral and has taken a loan.
- Bob has given collateral token allowance to `LoanExecutor`.
- Alice can now transfer the loan to Bob using `Loan.transfer`. Bob is the borrower now.
- Alice call `LoanExecutor.repayLoan`.
- Collateral tokens gets transferred to Bob and then gets pulled out from his wallet instantly.
- Collateral is sold to repay the loan.

### Recommendations
Considering removing the ability to transfer loan to other account. Restricting the ability to call `LoanExecutor.repayLoan` to actual borrower can also be a fix.

### Remark
Fixed. Now only the borrower can call `LoanExecutor.repayLoan`.

## [L-01] Uninitialized implementation contracts.
### Severity
Low

### Description
The Factory contract deploys proxy contracts for every new Vault and Loan. However the original implementation behind the proxies never gets initialized. Having uninitialized implementation contracts must be avoided to prevent any unforseen bugs.

```solidity
contract ValentFactory is IFactory {
  constructor(address _registry) {
    ...
    lenderVaultImpl = address(new LendersVault());
    loanImpl = address(new Loan());
  }
}
```

### Recommendations
Consider using OpenZeppelin's Initializable contract and its `_disableInitializer` function during the implementation deployment.

## [L-02] Lenders can update status and expiry of null collection.
### Severity
Low

### Description
In `LendersVault.createIntentCollection` the collection id of intents starts with `1` index. Hence the `0th` intent is an empty collection.

```solidity
collectionId = ++collectionCounter;     // starts with 1
_intents[collectionId] = _collection;
```

However the lender can change the `isEnabled` status and `expiresAt` expiry of the `0th` collection.

### Recommendations
Consider validating that the `collectionId` input in `extendIntentCollection` and `setIntentCollectionStatus` functions is non-zero.

### Remark
Fixed. The functions now validates a non-zero `collectionId`.

## [L-03] Lack of input validation in `LoanExecutor.createLoans`.
### Severity
Low

### Description
The `createLoans` function takes `LoanExecParams` as input but none of the input values get validated. Specially the `vault` and `swapper` address needs to be validated as the contract makes unprotected external calls to both of the addresses.

```solidity
    createdLoan = ILoan(vault.createLoan( ... ));
```
```solidity
    IERC20Decimals(borrowToken).safeTransfer(swapper, borrowTokenBal);
    ISwapper(swapper).swapOut(borrowToken, collToken, collSwapAmount, swapperData);
```

### Recommendations
Consider validating __all__ the elements of `LoanExecParams` input parameter.


## [L-04] Lack of input validation in `LoanExecutor.repayLoan`.
### Severity
Low

### Description
The `repayLoan` function takes `LoanRepayParams` as input but none of the input values get validated. Specially the `loan` and `swapper` addresses. 

Unprotected external calls are made to `loan` and `swapper` addresses.


### Recommendations
Consider validating __all__ the elements of `LoanRepayParams` input parameter.

### Remark
Partially Fixed. The function only validates `loan` parameter.


## [L-05] Use of default `0` value for `sqrtPriceLimitX96` parameter.
### Severity
Low

### Description
The `swapIn` and `swapOut` functions of `UniswapV3Swapper` contract hardcodes `0` as the `sqrtPriceLimitX96` value for Uniswap router's calls.

```solidity
    ISwapRouter(UNISWAP_V3_ROUTER).exactInputSingle(
      ISwapRouter.ExactInputSingleParams({
        tokenIn: tokenIn,
        tokenOut: tokenOut,
        fee: fee,
        recipient: msg.sender,
        deadline: block.timestamp + 1,
        amountIn: amountIn,
        amountOutMinimum: amountOutMin,
        sqrtPriceLimitX96: 0
      })
    );
```

The [Uniswap docs](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#swap-input-parameters) explicitly mentions that 

> `sqrtPriceLimitX96`: In production, this value can be used to set the limit for the price the swap will push the pool to, which can help protect against price impact or for setting up logic in a variety of price-relevant mechanisms.

Hard coding 0 value can lead to unforseen scenarios.

### Recommendations
Consider taking the `sqrtPriceLimitX96` value as a user input in the `swapperData` parameter instead of hardcoding it to 0.

### Remark
Fixed as recommended.


## [L-06] Revert on zero value token approvals.
### Severity
Low

### Description
The `swapIn` function of `UniswapV2Swapper` and `UniswapV3Swapper` contracts performs an unnecessary `tokenIn.safeDecreaseAllowance` operation after performing the token swap.

```solidity
  function swapIn(address tokenIn, address tokenOut, uint256 amountIn, bytes calldata swapperData) external {
    uint256 amountOutMin = abi.decode(swapperData, (uint256));

    IERC20Decimals(tokenIn).safeIncreaseAllowance(UNISWAP_V2_ROUTER, amountIn);

    IUniswapV2Router02(UNISWAP_V2_ROUTER).swapExactTokensForTokens(
      amountIn, amountOutMin, getPathForToken(tokenIn, tokenOut), msg.sender, block.timestamp + 1
    );

    /// @dev decrease allowance of tokenIn to UNISWAP_V2_ROUTER
    IERC20Decimals(tokenIn).safeDecreaseAllowance(
      UNISWAP_V2_ROUTER, IERC20Decimals(tokenIn).allowance(address(this), UNISWAP_V2_ROUTER)
    );
  }
```

After the `swapExactTokensForTokens` call there is no need to decrease allowance as all allowance will be cleared during the swap. 

So most likely the function will do `token.approve` with 0 as input. There are some tokens like BNB which reverts when the input approval amount is 0. Hence the swapper contracts are not compatible with those kind of tokens due to that extra approval statement.

### Recommendations
Consider removing the extra `tokenIn.safeDecreaseAllowance` operation.

### Remark
Fixed. The `swapIn` function is removed from `UniswapV2Swapper` and `UniswapV3Swapper` contracts.

---