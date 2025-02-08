# Introduction

A time-boxed security review of the **[Valent](https://diffusionlabs.xyz/)** protocol was done by **Akshay Srivastav**, with a focus on security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort with the goal of finding as many vulnerabilities as possible. I can not guarantee 100% security after the review, even if problems are found in contracts in my security review. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Akshay Srivastav**

Akshay Srivastav is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@akshaysrivastv](https://twitter.com/akshaysrivastv).

# About **Valent**

It is a liquidation-free, oracle-less, immutable, permission-less, and fixed-term/fixed-rate lending/borrowing protocol.


# Security Assessment Summary

- **valent-monorepo - [51f3a8257e9f32f5e166989559e6f7f5ddfbdcf0](https://github.com/Diffusion-Labs/valent-monorepo/tree/51f3a8257e9f32f5e166989559e6f7f5ddfbdcf0)**
- This is a follow up audit of my previous audit of Valent protocol which was performed in [Dec 2023](https://gist.github.com/akshaysrivastav/8f6fa4d1b4093e29951cbd3361e88f08).

### Scope

The following smart contracts were in scope of the audit:

- All contracts in [packages/contracts/src](https://github.com/Diffusion-Labs/valent-monorepo/tree/51f3a8257e9f32f5e166989559e6f7f5ddfbdcf0/packages/contracts/src)


---


# Findings Summary

The following number of issues were found, categorized by their severity:

| Severity    | Issues    |
| ----------- | :-------: |
| High        | 0         |
| Medium      | 1         |
| Low         | 6         |
| Info        | 0         |


---

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [M-01] | Borrow amount can be less than the expected borrow amount   | Medium   |
| [L-01] | `Loan.repay` can be executed infinitely with no token transfers      | Low      |
| [L-02] | Rounding issue in `Loan.repay`      | Low      |
| [L-03] | `Loan.seizeCollateral` can be called multiple times      | Low      |
| [L-04] | `createLoan` call can revert for tokens which revert on zero value transfers      | Low      |
| [L-05] | Lenders cannot always specify interest rate to be upto 100%      | Low      |
| [L-06] | Unused parameter      | Low      |


# Detailed Findings

## [M-01] Borrow amount can be less than the expected borrow amount.
### Severity
Medium

### Description
In `LendersVault.createLoan` the borrower only supplies the `collAmt` value that dictates how much collateral will the borrower lock. The amount of borrow tokens is not supplied as input directly.

The borrow token amount is calculated based upon `colAmt`, `strikePrice`, `interestRate` & `takerFee`. All these values can be chosen by the borrower except the `takerFee`.

In case the `takerFee` gets changed before the borrower's transaction gets executed on-chain then the borrower will receive different amount of borrow tokens than he originally expected.

If the `takerFee` is increased by protocol admins then the borrower will receive less amount of tokens than expected, and vice versa.


### Recommendations
Consider taking a `minOutAmount` parameter as input in the `createLoan` function and then validate that `borrowAmtReceived` value is not less than `minOutAmount`.


## [L-01] `Loan.repay` can be executed infinitely with no token transfers.
### Severity
Low

### Description
In case a loan is fully repaid, the `repay` function can still be executed any number of times with any input value.

The `previewRepay` will always return `(0, 0)` and `repay` will get executedd with no token transfers.


### Recommendations
Consider performing the `repayAmt == 0` check after the `previewRepay` call.
```diff
-    if (repayAmt == 0) revert Errors.ZeroAmtProvided();
     uint256 reclaimedAmt;
     (repayAmt, reclaimedAmt) = previewRepay(repayAmt);
+    if (repayAmt == 0) revert Errors.ZeroAmtProvided();
```


## [L-02] Rounding issue in `Loan.repay`.
### Severity
Low

### Description
The `repay` function looks like this:
```solidity
    function repay(uint256 repayAmt, bytes calldata data) external nonReentrant {
        ...
        (repayAmt, reclaimedAmt) = previewRepay(repayAmt);
        loanData.repaidAmt += repayAmt;
        IERC20Decimals(loanData.collToken).safeTransfer(loanData.borrower, reclaimedAmt);
        ...
    }

    function previewRepay(uint256 repayAmt) public view returns (uint256, uint256) {
        uint256 remainingAmt = loanData.borrowAmt - loanData.repaidAmt;
        if (repayAmt > remainingAmt) {
            repayAmt = remainingAmt;
        }
        return (repayAmt, (loanData.collAmt * repayAmt) / loanData.borrowAmt);
    }

```
It can be seen that the `reclaimedAmt` is always rounded down. This can be misused by a malicious lender to steal borrower's collateral. Lender can repay borrower's debt in small fractions so that the `reclaimedAmt` collateral amount will come out to be `0`. 

The borrower won't be able to repay the already repaid debt amount, hence cannot free up his collateral. That collateral can then be seized by lender after the loan expires.


<details>
<summary>
Click to expand Foundry Test (added into `test/Loan.t.sol`)
</summary>

```solidity
  function test_audit_poc() public {
    address lender = makeAddr("lender");
    address borrower = makeAddr("borrower");
    MockERC20 mockWbtc = new MockERC20("Wrapped Bitcoin", "WBTC", 8);
    mockUsdt.mint(lender, 40_000e18);

    // Create vault for lender
    vm.startPrank(lender);
    LendersVault vault = LendersVault(factory.deployVault());
    DataTypes.Intent[] memory intents = new DataTypes.Intent[](1);
    intents[0] = DataTypes.Intent({
      strikePrice: 40000e18,    // 1 wbtc = 40,000 usdt
      interestRate: 2e16,       // 2%
      duration: 15 days
    });
    DataTypes.IntentCollection memory collection = DataTypes.IntentCollection({
      collToken: address(mockWbtc),
      borrowToken: address(mockUsdt),
      minSingleLoanAmt: 0,
      maxSingleLoanAmt: type(uint256).max,
      expiresAt: block.timestamp + 15 days,
      isEnabled: true,
      intents: intents
    });
    uint256 collectionId = vault.createIntentCollection(collection);
    mockUsdt.transfer(address(vault), 40_000e18);
    vm.stopPrank();

    // Borrow
    mockWbtc.mint(borrower, 1e8);
    vm.startPrank(borrower);
    DataTypes.BorrowParams memory borrowParams = DataTypes.BorrowParams({collectionId: collectionId, intentId: 0, collAmt: 1e8});
    mockWbtc.approve(address(vault), 1e8);
    Loan loan = Loan(vault.createLoan(borrowParams, ""));
    vm.stopPrank();

    assertEq(mockWbtc.balanceOf(address(loan)), 1e8);     // WBTC got locked in Loan contract
    assertEq(mockUsdt.balanceOf(borrower), 39_120e18);    // Borrower got ~39K USDT
    uint vaultUsdtBalIni = mockUsdt.balanceOf(address(vault));

    // Lender repays maliciously
    mockUsdt.mint(lender, 1000e18);
    vm.startPrank(lender);
    mockUsdt.approve(address(loan), 1000e18);
    uint gas = gasleft();
    for(uint i; i < 2e3; i++) {
      loan.repay(4e14 - 1, "");
    }
    console.log("gas used", gas - gasleft());                            // ~30M as Eth Mainnet
    ( ,,,,,, uint256 repaidAmt, ) = loan.loanData();
    uint vaultUsdtBalFin = mockUsdt.balanceOf(address(vault));
    assertEq(vaultUsdtBalFin - vaultUsdtBalIni, 799999999999998000);     // 0.8 USDT (e18) repaid amount goes to lender's vault
    assertEq(mockWbtc.balanceOf(address(loan)), 1e8);                    // Still full collateral 1 WBTC is present in Loan contract
    assertEq(repaidAmt, 799999999999998000);                             // 0.8 USDT is considered as repaid
  }
```

</details>


The attacker is only limited by gas cost of the network. In case of a chain with cheap gas fees, a low decimal collateral token and a high decimal borrow token this attack may become more favourable to perform.

## [L-03] `Loan.seizeCollateral` can be called multiple times.
### Severity
Low

### Description
The `Loan.seizeCollateral` function can be called multiple times by donating any amount of collateral tokens to the `Loan` contract. The function only validates that collateral token balance of `Loan` contract must be non-zero, this can be easily bypassed.


## [L-04] `createLoan` call can revert for tokens which revert on zero value transfers.
### Severity
Low

### Description
The `createLoan` function transfer the taker + maker fee to registry. In case the fees are set as `0` and the borrow token is a token which reverts on zero value transfers, then the `createLoan` call will get reverted.


## [L-05] Lenders cannot always specify interest rate to be upto 100%.
### Severity
Low

### Description
The Valent protocol intends to allow lenders to specify the interest rate in their intents to be upto 100%.

But the `_getBorrowAmountMinusPremFee` function looks like this:

```solidity
    function _getBorrowAmountMinusPremFee(uint256 loanAmount, uint256 interestRate)
        ...
    {
        uint256 totalPremium = (loanAmount * interestRate) / INTEREST_BASE_UNIT;
        uint256 takerFee = (totalPremium * registry.takerFee()) / INTEREST_BASE_UNIT;
        uint256 makerFee = (totalPremium * registry.makerFee()) / INTEREST_BASE_UNIT;
        totalPremium += takerFee;

        if (totalPremium > loanAmount) revert Errors.PremiumGtLoanAmt();

        borrowAmtReceived = loanAmount - totalPremium;
        takerPlusMakerFee = takerFee + makerFee;
    }
```
It can be seen that `totalPremium` cannot be greater than `loanAmount`, and `totalPremium` also depends upon the `takerFee`.

So in the worst case when `takerFee` is 100% the maximum acceptable `interestRate` will be 50%, way less than the actually intended cap of 100%.  

## [L-06] Unused parameter.
### Severity
Low

### Description
The `tokenOut` input parameter of `UniswapV3SwapperMultiRoute.swapOut` function is never used.


---