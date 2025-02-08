# Introduction

A time-boxed security review of the **BlueStaking** protocol was done by **Akshay Srivastav**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort with a goal of finding as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with the smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Akshay Srivastav**

Akshay Srivastav is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@akshaysrivastv](https://twitter.com/akshaysrivastv).

# About **BlueStaking**

It is a staking protocol where users bond funds that are then transferred to a multisig. Those funds are refunded to users when staking ends.


# Security Assessment Scope

The following smart contracts were in scope of the audit:
- **BlueStaking.sol**
- **BlueMagicToken.sol**

---


# Findings Summary

The following number of issues were found, categorized by their severity:

| Severity    | Issues    |
| ----------- | :-------: |
| High        | 0         |
| Medium      | 0         |
| Low         | 7         |
| Info        | 13        |


---

# Detailed Findings

## Low Severity
1. In the deployment script of `BlueStaking` the ownership of `ProxyAdmin` is not transferred to multisig. Hence the right to upgrade the contracts remains with the deployment wallet.
2. `BlueMagicToken` contract has the `increaseAllowance` & `decreaseAllowance` functions which are now deprecated and have been removed from OpenZeppelin's `ERC20`. More details [here](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/4583).
3. Outdated OpenZeppelin version is being used.
4. In `BlueStaking.initialize` validate that `_startTime >= block.timestamp` (or atleast is non-zero).
5. Fund manager can dilute the rewards. The deposited tokens of users goes to `fundManager`, the fund manager can then again deposit those token in the staking contract and claim rewards. This can be done repeatedly.
6. `BlueStaking._revertIfMinAmount` function should be marked as `pure` to silence the compiler warning.
7. In `depositToken` & `calculateReward` functions there is no need to check that `amount > 0` as the `_revertIfMinAmount` already validates that.

## Informational
1. The `BlueMagicToken` contract can have multiple `MINTER_ROLE` and `DEFAULT_ADMIN_ROLE` which seems unncessary. Consider implementing a simple `minter` and `owner` addresses.
2. `BlueMagicToken`'s `lock` and `unlock` functions can be combined into a single function.
3. Use solidity's `Error` types instead of strings while reverting.
4. Update license of solidity files. Currently uses `MIT`.
5. All the state variables of `BlueStaking` can be made `immutable` as they are not changed throughout the lifecycle of the contract. Immutables work with upgradable contracts as well.
6. `BlueStaking`: Since the `userDeposits` mapping is never used in the contract lifecycle there is no need to store that data onchain. The deposit amounts can be fetched by indexing the `Deposited` event.
7. There is no fee mechanism in the staking protocol.
8. There is no need to protect the `depositToken` function with `nonReentrant` modifier. You only make external calls to trusted contracts (Meth, Puff & BMT).
9. In the `depositToken` function the `safeTransferFrom` call should be performed at the end of function. This way the contract follows the secure check-effect-interaction pattern.
10. There in no upper cap on the tokens that can be submitted. Hence large amount of reward tokens can potentially get minted if the influx of deposits is huge.
11. In `BlueStaking` rename the `_revertIfMinAmount` function to `_revertIfNotMinAmount`.
12. The `_revertIfMinAmount` ensures that minimum deposit amount of `puff` token is `1e18`. In case the price of `puff` token increases then this can become a difficult requirement to fulfill for depositors.
13. `BlueStaking.totalDeposits` copies the entire `Deposit[]` into `memory`. In case the size of this array increases significantly then the gas usage of this function may exceed the block gas limit of the network.