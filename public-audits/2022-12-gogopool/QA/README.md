# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/565
1. Ocyticus and TokenggAVAX contracts inherits BaseAbstract but do not initializes the `version` parameter. Ideally the `version` should be passed in constructors/initializers so that its initialization does not get missed.

2. In the Vault contract, the transfer of funds is only allowed to network contracts, since Vault is non-upgradeable and intended to be future proof it should allow transfer of funds to any address.

3. Protocol docs for Ocyticus contract only mentions about pausing rights but unpausing is also possible.

4. In ERC20Upgradeable `_disableInitializers` should be invoked in the `constructor`, the same thing is also recommended by OpenZeppelin. 

5. Many contracts redundantly inherit already inherited contracts. This makes the code less readable.

    | Contract           | Redundant Inheritance | Already Inherited In                |
    | ---                | -----------           | -------                             |
    | ERC4626Upgradeable | Initializable         | ERC20Upgradeable                    |
    | TokenggAVAX        | Initializable         | ERC4626Upgradeable, BaseUpgradeable |

6. The `onlyGuardian` modifier in `ProtocolDAO.initialize` function can be omitted as the function do not take any input values and the initialization is time independent.

7. `onlyRegisteredNetworkContract` modifier has different meaning and context in different contracts. In `Storage` it includes registered contracts and `guardian` while in `BaseAbstract` it only includes registered contracts.

8. In `Staking` contract, for `increaseAVAXAssignedHighWater` and `resetAVAXAssignedHighWater` `onlySpecificRegisteredContract` modifier should be used instead of `onlyRegisteredNetworkContract` modifer. As both the functions are only invoked by a single contract (`MinipoolManager` and `ClaimNodeOp` respectively).

9. MinipoolManager - incorrect comments. As per the convention of the overall protocol 2% should be 0.02 ether.
     - L35
     - L194

10. MinipoolManager - typo in `@notice` of `canClaimAndInitiateStaking`.

11. The protocol [docs](https://multisiglabs.notion.site/Architecture-Protocol-Overview-4b79e351133f4d959a65a15478ec0121) mentions that the contracts do not have any storage variables as they follow the Storage Registration Technique for upgradeability.  But the contracts do have some storage variables as they inherit Base, ReentrancyGuard contracts. 

12. ERC4626Upgradeable - a comment [section](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/tokens/upgradeable/ERC4626Upgradeable.sol#L24) mentions IMMUTABLES while there are no immutable variables in the contract.

13. In Vault contract no getter function is present to read `allowedTokens` mapping.

14. TokenggAVAX.sol imports BaseUpgradeable.sol using statement `import "../BaseUpgradeable.sol";` which is not the recommended way for importing other solidity files as per the Solidity [docs](https://docs.soliditylang.org/en/v0.8.17/layout-of-source-files.html#syntax-and-semantics). This form is not recommended for use, because it unpredictably pollutes the namespace.
The ideal way should be:
    ```solidity
    import {BaseUpgradeable} from "../BaseUpgradeable.sol";
    ```
    Same is the case with ClaimNodeOp.sol, ClaimProtocolDAO.sol and some other contract files.

15. Unnecessary imports present:

    | Contract File       | Imported Object           |
    | ---                 | -----------               |
    | TokenggAVAX.sol     | ERC20Upgradeable          |
    | ClaimNodeOp.sol     | MinipoolManager, TokenGGP |
    | MinipoolManager.sol | TokenGGP                  |
    | ProtocolDAO.sol     | TokenGGP                  |
    | Vault.sol           | TokenGGP, WAVAX           |


16. Vault contract contains unnecessary errors which are never used - `InvalidNetworkContract`, `TokenTransferFailed` and `VaultTokenWithdrawalFailed`.

17. Basic input sanitation for access protected functions should be done. Sometimes these functions are triggered by offchain automated softwares (scripts) which can possibly misbehave and pass null values (0). These null values can cause significant damage to the protocol and its users.

18. The codebase still has `TODO` tags. It is recommended to resolve all the `TODO` tags before deploying the contracts on mainnet.