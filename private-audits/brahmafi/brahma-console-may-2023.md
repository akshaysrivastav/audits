# Introduction

A time-boxed security review of the **[Brahma Console](https://console.brahma.fi)** protocol was done by **Akshay Srivastav**, with a focus on the security aspects of the application's implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Akshay Srivastav**

Akshay Srivastav is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews. Reach out on Twitter [@akshaysrivastv](https://twitter.com/akshaysrivastv).

# About **Brahma Console**

Brahma Console is an orchestration layer designed to enhance the DeFi experience on smart contract wallets. The initial version is built on safe, with user-configurable automation/strategies for frequent DeFi interactions, all of which are completely on-chain and executed in a trustless manner with keepers from Gelato and Brahma.


# Security Assessment Summary

- **console-core - [58bf05320bc5405f36549ca786a317724241e2ee](https://github.com/Brahma-fi/console-core/tree/58bf05320bc5405f36549ca786a317724241e2ee)**
- **console-integrations    - [1fea2c1166ae2cedd74cdfa2986ff6d908fcda06](https://github.com/Brahma-fi/console-integrations/tree/1fea2c1166ae2cedd74cdfa2986ff6d908fcda06)**

### Scope

The following smart contracts were in scope of the audit:

- All contracts in the `src` directory.


The following number of issues were found, categorized by their severity:

- High: 0 issues
- Medium: 3 issues
- Low: 16 issues

---

# Findings Summary

| ID     | Title                   | Severity |
| ------ | ----------------------- | -------- |
| [M-01] | Bots can executed trusted automations   | Medium   |
| [M-02] | Automation can be executed for a deregistered strategy   | Medium   |
| [M-03] | Updating the governance controlled variables can break all existing automations   | Medium   |
| [L-01] | Un-optimal `amountIn` calculation in CoWDCAStrategy and TrustedCoWDCAStrategy      | Low      |
| [L-02] | In WalletRegistry `WalletData.walletType` can be reset to `0`      | Low      |
| [L-03] | External bot subscriptions can be created even when there is not authorised bots present      | Low      |
| [L-04] | Race condition can arise among bots      | Low      |
| [L-05] | `isEmergencyPaused` flag is not considered in automation triggers      | Low      |
| [L-06] | Users can be forced to overpay in fee tokens      | Low      |
| [L-07] | Incorrect handling of `token.decimals` in PriceFeedManager      | Low      |
| [L-08] | Users cannot exit a strategy when the protocol is paused      | Low      |
| [L-09] | Users may need to wait for additional time in case the exchange on CowSwap fails      | Low      |
| [L-10] | Mismatch between developer comments and code implementation      | Low      |
| [L-11] | Incompatibility with non standard ERC20 tokens      | Low      |
| [L-12] | Users can deploy new console accounts and subscribe to new strategies when protocol is paused      | Low      |
| [L-13] | `GelatoBot.initTask` accepts `bytes32(0)` as the `currentTask`      | Low      |
| [L-14] | Hardcoded stale feed threshold for `PriceFeedManager.setTokenPriceFeed`      | Low      |
| [L-15] | Low-level calls does not validate existence of bytecode on target      | Low      |
| [L-16] | Optimistic reliance on chainlink price feeds      | Low      |

# Detailed Findings

# [M-01] Bots can executed trusted automations

## Severity
Medium

## Description
The `BrahRouter.executeAutomationViaBot` never checks the `trusted` flag. So currently when trusted = true & external = true any bot or keeper can execute the automation using `executeAutomationViaBot`. 
```solidity
    function executeAutomationViaBot(address _wallet, address _subAccount, address _strategy, bytes32 automationId)
        external
        claimExecutionFees(_wallet)
    {
        Authorizer._validateBot();

        Executor._executeAutomation(
            _wallet, _subAccount, _strategy, getAutomation(_wallet, _subAccount, _strategy, automationId)
        );

        emit BotExecution(_subAccount, _strategy, automationId);
    }
```

Also when `trusted` is true the keeper can execute the automation using the `executeAutomationViaBot` function, ideally only `executeTrustedAutomation` should be allowed to be invoked.

## Recommendations
Consider validating the `trusted` parameter value in both the `executeAutomationViaBot` and `executeTrustedAutomation` functions.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [M-02] Automation can be executed for a deregistered strategy

## Severity
Medium

## Description
`BrahRouter.executeTrustedAutomation` can be invoked by the keeper even after a strategy has been deregistered.
```solidity
    function executeTrustedAutomation(
        address _wallet,
        address _subAccount,
        address _strategy,
        Types.Executable[] memory _actionExecs
    ) external claimExecutionFees(_wallet) {
        Authorizer._validateConsoleKeeper();
        Authorizer._validateTrustedSubscription(_subAccount, _strategy);
        Executor._executeAutomation(_wallet, _subAccount, _strategy, _actionExecs);

        emit TrustedExecution(_subAccount, _strategy);
    }
```
The `executeTrustedAutomation` never validates the registered state of a strategy so in case a strategy gets deregistered it's automation can still be executed via `BrahRouter.executeTrustedAutomation` function.

## Recommendations
Consider enforcing that the strategy for which the automation is being performed is registered with `StrategyRegistry` contract.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [M-03] Updating the governance controlled variables can break all existing automations

## Severity
Medium

## Description
Updating the `botManager` and `brahRouter` parameters in AddressProvider using the `setBotManager` and `setBrahRouter` functions can impact the automation execution of all previous subAccounts/strategies. 

The `botManager` and `brahRouter` values are read at runtime by `BotManager.resolver`, `BotManager.execute`, `GelatoBot.resolver` & `GelatoBot.execute` functions. Any mismatch between deployment/initialization configuration and realtime configuration of a subAccount-strategy pair can lead to automation transactions getting reverted.

**GelatoBot.sol**
```solidity
    function resolver(address strategy, address wallet, address subAccount, bytes32 automationId)
        external
        view
        returns (bool, bytes memory)
    {
        return (
            BotManager(addressProvider.botManager()).resolver(strategy, wallet, subAccount, automationId),
            abi.encodeCall(this.execute, (strategy, wallet, subAccount, automationId))
        );
    }

    function execute(address strategy, address wallet, address subAccount, bytes32 automationId) external {
        if (!isBot(msg.sender)) revert UnauthorizedBot(msg.sender);

        BotManager(addressProvider.botManager()).execute(strategy, wallet, subAccount, automationId);
    }
```

## Recommendations
Consider making the `botManager` and `brahRouter` parameters immutables, or, consider caching them in Strategy parameters so that updating the paramters in AddressProvider does not impact any previous strategy-subAccount pair.

## Remark
Acknowledged with comment:
> Yes, we agree governance needs to be careful in case we upgrade botmanager/brahrouter

# [L-01] Un-optimal `amountIn` calculation in CoWDCAStrategy and TrustedCoWDCAStrategy. 

## Severity
Low

## Description
The `initStrategy` function of CoWDCAStrategy and TrustedCoWDCAStrategy calculates the `amountIn` value for each iteration as `amountIn = amountIn / iterations`. 

```solidity
    function initStrategy(
        ...
        uint256 amountIn,
        ...
    ) external returns (address) {
        ...
        amountIn = amountIn / iterations;

        StrategyParams memory params = StrategyParams({
            tokenIn: inputToken,
            tokenOut: outputToken,
            amountToSwap: amountIn,
            interval: interval,
            remitToOwner: remitToOwner,
            lastOrderEpoch: 0
        });
        ...
    }
```

This division can lead to precision loss. If the original `amountIn` was 100 and `iteration` was 3, then the `amountIn` for each iteration will come out to be `33`. The remaining 1 token amount will be swapped in the fourth iteration and depending upon the swap fee & gas price this dust amount may or may not be able to be swapped by bots, preventing the auto exit of the strategy. Failing automation swaps can also result in loss of ETH for bots/keeper.

## Recommendations
Consider validating that `amountIn % iterations == 0`.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-integrations/pull/9/).


# [L-02] In WalletRegistry `WalletData.walletType` can be reset to `0`

## Severity
Low

## Description
The `WalletRegistry.upgradeWalletType` function does not check whether the `_upgradablePaths` mapping value is non-zero. So if the function is invoked for a non-existing `_upgradablePaths` the wallet's `walletType` will be set to 0.
```solidity
    function upgradeWalletType() external {
        if (!isWallet(msg.sender)) revert WalletDoesntExist(msg.sender);

        uint8 fromWalletType = _walletDataMap[msg.sender].walletType;
        _setWalletType(msg.sender, _upgradablePaths[fromWalletType]);

        emit WalletUpgraded(msg.sender, fromWalletType, _upgradablePaths[fromWalletType]);
    }
```
Due to this the `isWallet` function will return `false` for an existing wallet.

## Recommendations
Consider validating that the `_upgradablePaths` map value for a `walletType` must be non-zero.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/) with comment:
> FIXED, removed upgrade feature, replaced with option to deregister wallet and register again with a different wallet type


# [L-03] External bot subscriptions can be created even when there is not authorised bots present

## Severity
Low

## Description
`BotManager.createTask` does not revert when `externalTask` is true but no `authorizedBots` are present.

```solidity
    function createTask(bool externalTask, address strategy, address wallet, address subAccount, bytes32 automationId)
        external
        returns (bytes32 currentTask)
    {
        ...
        if (externalTask) {
            uint256 len = authorizedBots.length();
            if (len > 0) {
                uint256 idx = 0;
                do {
                    (address _bot,) = authorizedBots.at(idx);
                    IBot(_bot).initTask(strategy, wallet, subAccount, automationId, currentTask);
                    unchecked {
                        ++idx;
                    }
                } while (idx < len);
            }
        }
        ...
    }
```
Due to this the execution of `BotManager.createTask` passes successfully even when `externalTask` is true but no `authorizedBots` are present

## Recommendations
Consider validating that the `authorizedBots.length()` value must be non-zero.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).

# [L-04] Race condition can arise among bots.

## Severity
Low

## Description
As all automations are open to be invoked by any bot, multiple bots can try to execute the same automation simultaneously which may result in only one bot's txn getting executed and the rest failing, resulting in loss of gas funds for bots.

## Recommendations
Consider enforcing an offchain priority queue for bots so that lower priority bots only tries to execute an automation if all higher priority bots fails to do so.

## Remark
Acknowledged with comment:
> We are aware of this race condition but we need this in order to maintain redudancy and quick execution, very useful in cases like preventing liquidations.

# [L-05] `isEmergencyPaused` flag is not considered in automation triggers.

## Severity
Low

## Description
The `Executor._checkTrigger` does not take `isEmergencyPaused` flag into consideration. When paused the triggers will return `true` tricking the bots/keeper to submit failing automation txns.

```solidity
    function _checkTrigger(address _subAccount, address _strategy, bytes32 _automationId)
        internal
        view
        returns (bool)
    {
        IStrategy strategy = IStrategy(_strategy);
        Types.Executable[] memory triggerCheck = strategy.getTriggerExecs(_automationId, _subAccount);
        uint256 triggerLen = triggerCheck.length;

        if (triggerLen == 0) {
            revert InvalidTriggers();
        } else {
            uint256 idx = 0;
            do {
                ...
            } while (idx < triggerLen);
        }
        return true;
    }
```


## Recommendations
Consider adding this statement in the `_checkTrigger` function:
```solidity
    if (isEmergencyPaused) revert EmergencyPaused();
```

## Remark
Acknowledged with comment:
> Bots like gelato and chainlink simulate txns before submitting, and do not submit failing txns.


# [L-06] Users can be forced to overpay in fee tokens.

## Severity
Low

## Description
`FeePayer._buildFeeExecutable` tries to collect fees from safes for the automated txn. These fee amounts depend upon the `gasprice` of txns which is controlled by bots/keeper. It is possible that the users may overpay in terms of fee tokens due to bots submitting txns with excessively high gas price.

```solidity
    function _buildFeeExecutable(uint256 gasUsed, address feeToken)
        internal
        view
        returns (uint256, address, Types.Executable memory)
    {
        address recipient = _addressProvider.fundManager();
        if (feeToken == ETH) {
            uint256 totalFee = (gasUsed + GAS_OVERHEAD_NATIVE) * tx.gasprice;
            totalFee = _applyMultiplier(totalFee);
            return (totalFee, recipient, TokenTransfer._nativeTransferExec(recipient, totalFee));
        } else {
            uint256 totalFee = (gasUsed + GAS_OVERHEAD_ERC20) * tx.gasprice;
            // Convert fee amount value in fee token
            uint256 feeToCollect =
                PriceFeedManager(_addressProvider.priceFeedManager()).getTokenXPriceInY(totalFee, ETH, feeToken);
            feeToCollect = _applyMultiplier(feeToCollect);
            return (feeToCollect, recipient, TokenTransfer._erc20TransferExec(feeToken, recipient, feeToCollect));
        }
    }
```

## Recommendations
Consider implemeting some ways by which users can specify a range of acceptable gasprice values.

## Remark
Acknowledged with comment:
> We ackwnoledge this concern. Bots like gelato have an internal bidding and slashing mechanism where txn submitters stake GEL tokens and are incentivized to keep gas prices competitive or they get slashed.


# [L-07] Incorrect handling of `token.decimals` in PriceFeedManager.

## Severity
Low

## Description
In `PriceFeedManager.setTokenPriceFeed` the `tokenDecimals` are set to 18 when `token.decimals` call reverts. 

```solidity
    function setTokenPriceFeed(address token, address priceFeed) external {
        _onlyGov();
        uint8 tokenDecimals = 0;
        ...
        if (token != ETH) {
            try IERC20Metadata(token).decimals() returns (uint8 _decimals) {
                if (_decimals > 18) revert InvalidERC20Decimals(token);

                tokenDecimals = _decimals;
            } catch {
                tokenDecimals = 18;
            }
        } else {
            tokenDecimals = 18;
        }
        ...
    }
```

This can cause issues in the calculation of output amount of `getTokenXPriceInY`. Essentially the contract is treating a token to be of 18 decimals while it is not.

```solidity
    function getTokenXPriceInY(uint256 amount, address tokenX, address tokenY) external view returns (uint256) {
        if ((tokenX == ETH || tokenX == WETH) && (tokenY == ETH || tokenY == WETH)) {
            return amount;
        }

        (uint256 tokenXPrice, uint8 tokenXDecimals) = _getTokenData(tokenX);
        (uint256 tokenYPrice, uint8 tokenYDecimals) = _getTokenData(tokenY);

        /// NOTE: returned price is adjusted to decimals of `tokenY`
        return (uint256(tokenXPrice) * amount * 10 ** tokenYDecimals) / uint256(tokenYPrice) / 10 ** tokenXDecimals;
    }
```

## Recommendations
Consider reverting instead when the `token.decimals` call fails.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [L-08] Users cannot exit a strategy when the protocol is paused.

## Severity
Low

## Description
Users cannot invoke `exitStrategy` when `Executor.isEmergencyPaused` is set to true. The `IStrategy.exitStrategy` internally calls the `Executor._executeOnSubAccount` which reverts when protocol is paused. Note, as all funds are held in Safe wallets they can be pulled out anytime by the Safe owner regardless of protocol paused state.

## Recommendations
If possible, consider implementing a way for users to exit from strategies when protocol is paused.

## Remark
Acknowledged with comment:
> Yes, In case we have to pause due to a bug/exploit, we do not want to create an exception here to allow user to exit. Users still own the main safe and by extension the subaccount. They can execute txn to transfer assets from subaccount to mainsafe or other wallets anytime they wish to, We have added a method to allow user to rescue stale subaccount.


# [L-09] Users may need to wait for additional time in case the exchange on CowSwap fails. 

## Severity
Low

## Description
The `CoWDCAStrategy` tries to swap tokens on every interval. The Cowswap's swap is a non-atomic operation, only the `OrderPlacement` event is emitted onchain and the actual swap happens via offchain relayers. 

On every automation (regardless of success or failure of a swap) the `CoWDCAStrategy.initSwap` function is invoked which updates the `StrategyParams.lastOrderEpoch` parameter. 
```solidity
    function initSwap() external {
        SubscriptionRegistry subscriptionRegistry = SubscriptionRegistry(addressProvider.subscriptionRegistry());
        StrategyParams memory params =
            abi.decode(subscriptionRegistry.retrieveSubData(msg.sender, strategyAddress), (StrategyParams));
        params.lastOrderEpoch = block.timestamp;
        subscriptionRegistry.updateSubData(msg.sender, strategyAddress, abi.encode(params));
    }
```

In case the swap becomes unsuccessful the strategy user and bots will need to wait for the next time interval to perform the same token swap again.

## Recommendations
Consider implementing ways to retry the swap again as soon as the offchain swap relay fails.

## Remark
Acknowledged as expected behaviour.

# [L-10] Mismatch between developer comments and code implementation.

## Severity
Low

## Description
`DCACoWAutomation.canInitSwap` does not validates all conditions as described in the developer comments.
```solidity
    /**
     * @notice Initiate Swap Check
     *
     * @dev Checks if a DCA swap can be executed
     *     Checks for existing active orders
     *     Checks if subAccount has enough balance to execute swap
     *     Checks if enough time has passed since last swap
     *
     *
     * @param subAccount address of subAccount
     * @param inputToken address of inputToken
     * @param interval DCA interval in seconds
     * @param lastSwap timestamp of last swap
     * @return Boolean wether a swap can be initiated
     */
    function canInitSwap(address subAccount, address inputToken, uint256 interval, uint256 lastSwap)
        external
        view
        returns (bool)
    {
        if (hasZeroBalance(subAccount, inputToken)) {
            return false;
        }
        return ((lastSwap + interval) < block.timestamp);
    }
```
The comments mention a check for existing active orders which is not implemented in the code.

## Recommendations
Consider fixing the the dev comments or add the expected functionality.

## Remark
Fixed natspec in [PR](https://github.com/Brahma-fi/console-integrations/pull/9/).


# [L-11] Incompatibility with non standard ERC20 tokens

## Severity
Low

## Description
The Brahma console protocol might not be compatible with non-standard ERC20 tokens, like fee-on-transfer tokens. 

For instance, the strict balance equality checks in `BrahRouter.claimExecutionFees` modifier can cause revert for fee on transfer tokens.
```solidity
    modifier claimExecutionFees(address _wallet) {
        uint256 startGas = gasleft();
        _;
        if (feeMultiplier > 0) {
            ...
            if (feeToken != ETH) {
                uint256 initialBalance = IERC20(feeToken).balanceOf(recipient);
                _executeSafeERC20Transfer(_wallet, feeTransferTxn);
                if (IERC20(feeToken).balanceOf(recipient) - initialBalance < feeAmount) {
                    revert UnsuccessfulFeeTransfer(_wallet, feeToken);
                }
            } else {
                uint256 initialBalance = recipient.balance;
                Executor._executeOnWallet(_wallet, feeTransferTxn);
                if (recipient.balance - initialBalance < feeAmount) {
                    revert UnsuccessfulFeeTransfer(_wallet, feeToken);
                }
            }
        }
    }
```

It should be noted that USDT is a widely used ERC20 token which also has the ability to charge fee on every token transfer (the actual amount received by receiver will be less than what was sent).

## Recommendations
Consider removing the strict balance equality check from the `claimExecutionFees` modifier.

## Remark
Acknowledged with comment:
> We dont wish to support fee-on-transfer erc20 tokens


# [L-12] Users can deploy new console accounts and subscribe to new strategies when protocol is paused

## Severity
Low

## Description
After `Executor.isEmergencyPaused` has been set to `true`, users can still deploy new console accounts and subscribe to strategies. New subscription can be done by either passing an empty `Types.TokenRequest` array or by passing 0 as the `Types.TokenRequest.amount` to the `BrahRouter.requestSubAccountFunds` function.

```solidity
    function requestSubAccountFunds(address _wallet, address _subAccount, Types.TokenRequest[] memory _tokenRequests)
        external
    {
        ...
        uint256 tokenRequestLen = _tokenRequests.length;
        if (tokenRequestLen > 0) {
            uint256 idx = 0;
            do {
                if (_tokenRequests[idx].amount != 0) {
                    Types.Executable memory tokenTransfer = TokenTransfer._erc20TransferExec(
                        _tokenRequests[idx].token, _subAccount, _tokenRequests[idx].amount
                    );
                    _executeSafeERC20Transfer(_wallet, tokenTransfer);
                }
                unchecked {
                    ++idx;
                }
            } while (idx < tokenRequestLen);
        }
    }

```

## Recommendations
Consider preventing the ability of users to create new console accounts and subscribe to strategies when protocol is paused.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [L-13] `GelatoBot.initTask` accepts `bytes32(0)` as the `currentTask`

## Severity
Low

## Description
`GelatoBot.initTask` does not validate that the `currentTask` value returned from `automate.createTask` call must not be equal to `bytes32(0)`.

```solidity
    function initTask(address strategy, address wallet, address subAccount, bytes32 automationId, bytes32 internalId)
        external
    {
        ...
        bytes32 currentTask = automate.createTask(
            address(this), // _execAddress
            abi.encodeCall(this.execute, (strategy, wallet, subAccount, automationId)), // _execDataOrSelector
            moduleData, // _moduleData
            address(0) // _feeToken
        );
        taskIds[internalId] = currentTask;
    }
```

## Recommendations
Consider adding this statement:
```solidity
    require(currentTask != bytes32(0));
```

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [L-14] Hardcoded stale feed threshold for `PriceFeedManager.setTokenPriceFeed`

## Severity
Low

## Description
The `PriceFeedManager.setTokenPriceFeed` validates the round data with a hardcoded `DEFAULT_STALE_FEED_THRESHOLD`. Every chainlink feed can have a different heartbeat or update frequency.

In case the update frequency of the chainlink feed is greater than `DEFAULT_STALE_FEED_THRESHOLD` then the `setTokenPriceFeed` can revert while setting up the price feed.

```solidity
    function setTokenPriceFeed(address token, address priceFeed) external {
        ...
        try IAggregatorV3(priceFeed).latestRoundData() returns (
            uint80 roundId, int256 price, uint256, uint256 updatedAt, uint80 answeredInRound
        ) {
            _validateRound(roundId, answeredInRound, price, updatedAt, DEFAULT_STALE_FEED_THRESHOLD);
        } catch {
            revert InvalidPriceFeed(priceFeed);
        }
        ...
    }
    function _validateRound(
        uint80 roundId,
        uint80 answeredInRound,
        int256 _latestPrice,
        uint256 _lastUpdatedAt,
        uint256 _staleFeedThreshold
    ) internal view {
        if (_latestPrice <= 0) revert InvalidPriceFromRound(roundId);
        if ((block.timestamp - _lastUpdatedAt > _staleFeedThreshold) || (answeredInRound < roundId)) {
            revert PriceFeedStale();
        }
    }
```

## Recommendations
Ideally a non-zero `staleFeedThreshold` parameter should be taken as the input in the  `PriceFeedManager.setTokenPriceFeed` function.

## Remark
Fixed in [PR](https://github.com/Brahma-fi/console-core/pull/54/).


# [L-15] Low-level calls does not validate existence of bytecode on target.

## Severity
Low

## Description
The low level calls in Executor (`staticcall` in `_executeStatic` & `call` in `_execute`) does not validate if bytecode exist on the `target` address. Low level calls to non-existing contracts also returns `true` as boolean.

This issue can trick Executor into believing a call to be successful when it is not.

## Recommendations
Consider validating the presence of bytecode on the `target` address on which the low level call is performed.

## Remark
Acknowledged with comment:
> We are aware about this possibility. `isContract` check is quite expensive as it loads the complete EXTCODE of an address before it checks size. All addresses being sent to this call are sent by trusted contracts so we chose to skip this check in interest of gas savings.


# [L-16] Optimistic reliance on chainlink price feeds

## Severity
Low

## Description
The Brahma console protocol heavily relies on working chainlink price feeds. If chainlink feeds go down or report stale/invalid data then all automations (trusted/untrusted) of Brahma console protocol will suffer DoS situation. The `PriceFeedManager.getTokenXPriceInY` is invoked in all automations which will cause the DoS.

## Recommendations
Consider implementing fallback price feeds in case chainlink feed goes down.

## Remark
Acknowledged with comment:
> Yes we agree, although less, the chances of DoS are non zero.

