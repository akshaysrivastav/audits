# Introduction

A time-boxed security review of the **[TITAN X](https://www.titanx.win/)** protocol was done by **Akshay Srivastav**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Akshay Srivastav**

Akshay Srivastav is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@akshaysrivastv](https://twitter.com/akshaysrivastv).

# About **TITAN X**

The sole design purpose for TITAN X is to reduce supply, add programatic buy pressure through smart contracts & drive demand to the ecosystem through various avenues and game theory mechanics.


# Security Assessment Summary

- **ttx_private - [475ce087b246722959f8803a31726e3000119fb5](https://github.com/jakesharpe777/ttx_private/tree/475ce087b246722959f8803a31726e3000119fb5)**
- **ttx_buyandburn_private    - [91bf474aaf9481a4779f94b00b753eaa45495d51](https://github.com/jakesharpe777/ttx_buyandburn_private/tree/91bf474aaf9481a4779f94b00b753eaa45495d51)**

### Scope

The following smart contracts were in scope of the audit:

#### ttx_private repo
- TITANX.sol
- StakeInfo.sol
- MintInfo.sol
- BurnInfo.sol
- GlobalInfo.sol
- OwnerInfo.sol

#### ttx_buyandburn_private repo
- BuyAndBurn.sol


---


# Findings Summary

The following number of issues were found, categorized by their severity:

| Severity    | Issues    |
| ----------- | :-------: |
| High        | 4         |
| Medium      | 5         |
| Low         | 2         |
| Info        | 3         |


---

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [H-01] | Any user's liquid tokens, stakes and mints can be burned due to open burn functions   | High   |
| [H-02] | Gaming the protocol incentives during burning   | High   |
| [H-03] | Loss of protocol TITX share when mints are burned   | High   |
| [H-04] | No slippage protection when selling Weth for Titan   | High   |
| [M-01] | Missing dailyUpdate modifier   | Medium   |
| [M-02] | Delay in transaction execution can lead to less amount of tokens getting minted to users than they expect   | Medium   |
| [M-03] | ETH received by TITANX contract are lost forever   | Medium   |
| [M-04] | BuyAndBurn swap mechanism can fail if contract holds large amount of WETH   | Medium   |
| [M-05] | Frontrunning the initial liquidity creation   | Medium   |
| [L-01] | Incorrect integration with ERC165 enabled contracts      | Low      |
| [L-02] | ERC20 increaseAllowance and decreaseAllowance functions have been deprecated      | Low      |
| [I-01] | Misspelling      | Info      |
| [I-02] | Misleading comments      | Info      |
| [I-03] | Unused structs      | Info      |


# Detailed Findings
## [H-01] Any user's liquid tokens, stakes and mints can be burned due to open burn functions.
### Severity
High

### Description
The TITANX smart contract has these functions:
- burnTokens
- burnTokensToPayAddress
- burnStake
- burnStakeToPayAddress
- burnMint
- burnMintToPayAddress

These functions are intended to be called by other DeFi protocols to incentivize burning of TITX tokens/stakes/mints of a user.

However, since these are openly accessible functions an attacker can burn any account's tokens/stakes/mints leading to huge loss of funds for users.

#### Attack Scenario
- An attacker calls `burnTokens` for the TITX AMM pool, let's say WETH-TITX pool.
- Entire TITX token balance of pool gets burned.
- Now attacker can sell a small amount of TITX tokens for the entire WETH balance of the pool.


### Recommendations
Consider adding access protections on these functions which should be controllable by users.

Eg, for `burnTokens` users can provide TITX token allowance to certain defi protocols and only these protocols should be able to burn tokens of particular users (upto the allowance limit). Additional mechanisms will be needed to protect burning of mints & stakes. 

### Remark
To fix this issue, allowances are introduced in the contract. Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [H-02] Gaming the protocol incentives during burning.
### Severity
High

### Description
The TITANX smart contract has these functions:
- burnTokens
- userBurnTokens
- burnStake
- userBurnStake
- burnMint
- userBurnMint

The `burnTokens` is intended to be called by defi protocols while `userBurnTokens` is intended to be called by users themselves (similar case for stake/mint). The `burnTokens` function provides additional rewards to defi protocols and users (upto 8% as per [docs](https://docs.titanx.win/titanx/titan-x/proof-of-burn-2.0#project-burning)).

Since `burnTokens` is an open function, the users will always call `burnTokens` instead of calling the intended `userBurnTokens` to earn additional rewards. Due to this the users will be able to harvest more rewards from the protocol than originally intended.

Note: the issue is similar for burning stakes and mints.

### Recommendations
Decent design level changes will be needed to completely resolve this issue. Some possible solutions could be:
- Add a whitelist of protocols and only they can burn assets of users.
- Start giving the 8% incentives to users as well (in the `userBurnTokens` function).

### Remark
Acknowledged with comment:
> The main objective is to burn huge supply, the small percentage reward is extra incentive to attract builders, we welcome anyone to build a dummy contract to burn their huge supply for percentage reward.

## [H-03] Loss of protocol TITX share when mints are burned.
### Severity
High

### Description
When a "mint" is claimed, an additional 1% of claim amount is minted to `s_genesisAddress`.

```solidity
    function claimMint(uint256 id) external dailyUpdate nonReentrant {
        uint256 reward = _claimMint(_msgSender(), id, MintAction.CLAIM);
        _mintReward(reward);
    }
    function _mintReward(uint256 reward) private {
        _mint(_msgSender(), reward);
        _mint(s_genesisAddress, (reward * 100) / PERCENT_BPS);
    }
```
This genesis wallet share does not get minted when the "mint" is burned by users. This leads to loss of protocol TITX share.

Ideally the end state of contract must be identical in these two scenarios:
1. Mint created -> Mint claimed -> Claimed TITX burned
2. Mint created -> Mint burned

But currently the outcome of these two scenario differs as in the second scenario protocol TITX share is not minted.

### Recommendations
Consider minting the protocol TITX share in mint burning functions as well.

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [H-04] No slippage protection when selling Weth for Titan.
### Severity
High

### Description
The `BuyAndBurn` contract has a public `buynBurn` function which sells the entire contract's WETH balance for Titan token. The swap implementation looks like this:

[BuyAndBurn.sol](https://github.com/jakesharpe777/ttx_buyandburn_private/blob/91bf474aaf9481a4779f94b00b753eaa45495d51/contracts/BuyAndBurn.sol#L300-L306)
```solidity
    function buynBurn() public nonReentrant {
        ...
        uint256 wethBalance = getWethBalance(address(this));
        uint256 feeFunds = s_feesBuyAndBurn;
        s_feesBuyAndBurn = 0;

        uint256 incentiveFee = (wethBalance * INCENTIVE_FEE) /
            INCENTIVE_FEE_PERCENT_BASE;
        IWETH9(WETH9).withdraw(incentiveFee);

        wethBalance -= incentiveFee;
        if (wethBalance != 0) s_totalWethBuyAndBurn += wethBalance - feeFunds;
        if (feeFunds != 0) s_totalWethFeesBuyAndBurn += feeFunds;

        _swapWETHForTitan(wethBalance);
        TransferHelper.safeTransferETH(payable(msg.sender), incentiveFee);
    }
    
    function _swapWETHForTitan(uint256 amountWETH) private {
        (int256 amount0, int256 amount1) = IUniswapV3Pool(s_poolAddress).swap(
            address(this),
            WETH9 < s_titxAddress,
            int256(amountWETH),
            WETH9 < s_titxAddress ? MIN_SQRT_RATIO + 1 : MAX_SQRT_RATIO - 1,
            ""
        );
        uint256 titan = WETH9 < s_titxAddress
            ? uint256(amount1 >= 0 ? amount1 : -amount1)
            : uint256(amount0 >= 0 ? amount0 : -amount0);
        s_totalTitanBuyAndBurn += titan;
        burnLPTitan();
        emit BoughtAndBurned(amountWETH, titan, msg.sender);
    }
```
It can be seen that the contract performs the swap directly on the uniswap pool rather than the uniswap router contract. The `IUniswapV3Pool.swap` call does not provide any slippage protection for the caller.

Due to this, the above swap mechanism can be exploited by attackers/MEV searchers to gain profit from the swap.

#### Steps to attack
1. Increase the price of Titan in uniswap pool by buying Titan.
2. Call `buynBurn`. The protocol buys TItan at inflated prices.
3. Decrease the titan price back in uniswap pool by selling Titan and capture the profit.

### Recommendations
Consider using appropriate slippage parameters for the swap. This can done by either performing the slippage check in BuyAndBurn contract or using the uniswap's router contract. 

### Remark
Acknowledged with comment:
> The twap doesn't seem ideal, and it's less effective after couple months when we have a much thicker liquidity.
The workaround is we have a clone buynburn function which takes one arugument as amount, so in case we got issue with arbs, then we can buy in smaller amount using this backup function

Note: since the `buyAndBurn` are public functions they can still be used by MEV searchers as per their convenience. Hence the attack scenario can still materialize.

## [M-01] Missing `dailyUpdate` modifier.

### Severity
Medium

### Description
The `TITANX.userBurnTokens` is missing the `dailyUpdate` modifier. This modifier is responsible for progressing the daily increment/decrement of crucial state variable of contract like ETH cost for miners, TITX mintable per mint, etc.

[TITANX.sol](https://github.com/jakesharpe777/ttx_private/blob/475ce087b246722959f8803a31726e3000119fb5/contracts/TITANX.sol#L938-L949)
```solidity
    function userBurnTokens(uint256 amount) public nonReentrant {
        ...
    }
```

### Recommendations
Consider adding the `dailyUpdate` modifier to `userBurnTokens` function.

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [M-02] Delay in transaction execution can lead to less amount of tokens getting minted to users than they expect.
### Severity
Medium

### Description
The `startMint` function is used by the users to mint TITX tokens. The mint factors (ETH costs, TITX mintable per day, etc) changes every day as per the design of protocol.

Since the `startMint` function does not allow users to provide a minimum output amount of TITX tokens that they must receive, it is possible that amount that gets minted to users differs significantly than what they originally expected.

In case a user's txn gets stuck in Ethereum mempool due to low gas price provided and the current day changes, then once the txn gets executed the user will receive significantly less number of tokens than they originally expected. It is not rare for ethereum transactions to stay in pending state for hours, days or even weeks.

```solidity
    function startMint(
        uint256 mintPower,
        uint256 numOfDays
    ) external payable nonReentrant dailyUpdate { ... }
```

### Recommendations
Consider adding `deadline` and `minOutput` parameters to `startMint` function.

### Remark
Acknowledged with comment:
> We decided to let it run through, tRank bonus would cover their loss, the main goal is to get in as early as possible, this is by design and adding a revert on day tickover would hurt user experience more than just letting it go through.

## [M-03] ETH received by TITANX contract are lost forever.
### Severity
Medium

### Description
The `TITANX` contract contains a `receive` function. This means that the contract accepts plain ETH transfers.
```solidity
    receive() external payable {}
```
However there is no function in the contract to pull out those received ETH.

This breaks the default behaviour of Solidity which prevent accounts from sending ETH to a contract which has no `receive` or `payable fallback` functions. This is done to prevent ETH from getting stuck in smart contracts.

Note that, the ETH received via the minting process are handled separately by the contract, that amount is stored in `s_undistributedEth` and is distributed correctly.


### Recommendations
Consider removing the `receive` function.

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [M-04] BuyAndBurn swap mechanism can fail if contract holds large amount of WETH.
### Severity
Medium

### Description
The current swap implementation of `BuyAndBurn` contract tries to sell its entire WETH balance.
```solidity
    function buynBurn() public nonReentrant {
        ...
        uint256 wethBalance = getWethBalance(address(this));
        ...

        _swapWETHForTitan(wethBalance);
        TransferHelper.safeTransferETH(payable(msg.sender), incentiveFee);
    }
```

There can exist scenarios in which the uniswap pool liquidity may not be sufficient enough to support a large WETH to Titan swap. In that case all the `buynBurn` txns will fail until the pool liquidity recovers.

Currently there is no mechanism to swap partial amount of WETH tokens held by the `BuyAndBurn` contract.

### Recommendations
Consider enabling a mechanism in BuyAndBurn to sell partial amounts of WETH tokens to the uniswap pool.

### Remark
Fixed in commit `caff7224c55a49962974047090d34ad8dcd9cd6e`.


## [M-05] Frontrunning the initial liquidity creation.
### Severity
Medium

### Description
The BuyAndBurn contract contains a one time function `createInitialLiquidity`. This function is intended to create the Titan-Weth pool on UniswapV3 and provide initial liquidity.

It is possible that the `createInitialLiquidity` call can be frontrunned. As soon as the Titan token is deployed, someone else can create a UniswapV3 pool for Titan-Weth pair and initialize it.

The initial liquidity creation looks like this: 
```solidity
    function createInitialLiquidity() public {
        ...
        _createPool();
        ...
        _mintPosition();
    }
    function _createPool() private {
        (address token0, address token1, , ) = _getTokensConfig();
        s_poolAddress = INonfungiblePositionManager(NONFUNGIBLEPOSITIONMANAGER)
            .createAndInitializePoolIfNecessary(
                token0,
                token1,
                POOLFEE1PERCENT,
                WETH9 < s_titxAddress
                    ? INITIAL_SQRTPRICE_WETH_TITX
                    : INITIAL_SQRTPRICE_TITX_WETH
            );
    }
    function _mintPosition() private {
        ...
        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams({
                token0: token0,
                token1: token1,
                fee: POOLFEE1PERCENT,
                tickLower: MIN_TICK,
                tickUpper: MAX_TICK,
                amount0Desired: amount0Desired,
                amount1Desired: amount1Desired,
                amount0Min: 0,                      // @audit
                amount1Min: 0,                      // @audit
                recipient: address(this),
                deadline: block.timestamp + 600
            });

        (uint256 tokenId, uint256 liquidity, , ) = INonfungiblePositionManager(
            NONFUNGIBLEPOSITIONMANAGER
        ).mint(params);
        ...
    }
```

It can be seen that the contract passes `0` as the `amount0Min` and `amount1Min` `MintParams` parameters, which is incorrect and not recommended.

In case the pool is already created and initialized, the uniswap pool will return back some amounts of Weth and Titan tokens to BuyAndBurn contract. Since the contract has no mechanism to implicitly handle the returned tokens, these will happen:
- The returned Weth will be subsequently used to buy Titan tokens. Hence the Weth which was intended to be supplied into liquidity will be used for buying Titan.
- The returned Titan will be burned in the subsequent `burnLPTitan` call.

Both the scenarios are unoptimal and were never originally intended to happen. 


### Recommendations
Consider implementing these fixes:
- Try to handle the scenario in which a Weth-Titan pool is already created. The pool can be initialized with absurd `sqrtPriceX96` initial price.
- Consider passing appropriate `amount0Min` and `amount1Min` values.
- Consider optimally handling the scenario in which some amount of tokens get returned back by the pool after liquidity addition.

### Remark
Fixed in commit `caff7224c55a49962974047090d34ad8dcd9cd6e` with comment:
> We moved out the pool creation logic into a new function. we will call this init pool right after the deployment and setting titx address to the buy and burn contract.
added the minimum amount when adding liquidity.

## [L-01] Incorrect integration with ERC165 enabled contracts.
### Severity
Low

### Description
In the current `TITANX._burnbefore` implementation the contract calls `supportsInterface` method with custom interface id without first checking the interface id of ERC165 itself.

```solidity
    function _burnbefore(
        ...
    ) private view {
        ...
        //Only supported contracts is allowed to call this function
        if (!IERC165(_msgSender()).supportsInterface(type(ITitanOnBurn).interfaceId))
            revert TitanX_NotSupportedContract();
    }
```

This integration implementation is incorrect. The correct way of integrating with ERC165 can be seen [here](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-165.md#how-to-detect-if-a-contract-implements-erc-165).

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [L-02] ERC20 `increaseAllowance` and `decreaseAllowance` functions have been deprecated.
### Severity
Low

### Description
After a recent incident the increase and decrease allowance functions have been deprecated and have been removed from several erc20 repositories. The functions have potential to lead to phishing attacks. See [this](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/4583) for further details. 

Consider removing these functions.

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.


## [I-01] Misspelling.
### Severity
Informational

### Description
Functions:
- [_updateUserBurnCylceClaimIndex](https://github.com/jakesharpe777/ttx_private/blob/475ce087b246722959f8803a31726e3000119fb5/contracts/GlobalInfo.sol#L273) -> _updateUserBurnCycleClaimIndex

Events
- [ProtocolFeeRecevied](https://github.com/jakesharpe777/ttx_private/blob/475ce087b246722959f8803a31726e3000119fb5/contracts/TITANX.sol#L446) -> ProtocolFeeReceived

### Remark
Partially fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`. The event is still misspelled.


## [I-02] Misleading comments.
### Severity
Informational

### Description
The comment [here](https://github.com/jakesharpe777/ttx_private/blob/475ce087b246722959f8803a31726e3000119fb5/contracts/GlobalInfo.sol#L14) states that:
> shareRate starts 1000 ether and increases capped at 2800 ether

The share rate actually starts from 800 ether.

### Remark
Fixed in commit `33ffdec0a98babc51ec9feff9124b4abb6e77ef3`.

## [I-03] Unused structs.
### Severity
Informational

### Description
The [SwapCallbackData](https://github.com/jakesharpe777/ttx_buyandburn_private/blob/91bf474aaf9481a4779f94b00b753eaa45495d51/contracts/BuyAndBurn.sol#L56-L59) struct in `BuyAndBurn` contract is unused.

### Remark
Fixed in commit `caff7224c55a49962974047090d34ad8dcd9cd6e`.


