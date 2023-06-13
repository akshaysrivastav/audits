# All my Security Audits, Reviews and Contributions


### Public Audits & Bug Bounties Stats
I participate on public audit platforms like Code4rena, Sherlock and Hats Finance. Till now I have :

- Participated in 20+ public audits
- Reported 50+ High and Medium severity bugs

### Top public audits

[comment]: <> (TODO - add Gravita report link)

| Audit Contest   |      Rank      |  Results |
|----------|:-------------:|------:|
| Ondo Finance |  1st | [link](https://code4rena.com/contests/2023-01-ondo-finance-contest) |
| Gravita Protocol |    1st   |   [link](https://github.com/Gravita-Protocol/Gravita-SmartContracts/issues/238) |
| Aragon Protocol | 4th |    [link](https://code4rena.com/contests/2023-03-aragon-protocol-contest) |
| Pool Together | 4th |    [link](https://code4rena.com/contests/2022-12-pooltogether-contest) |
| Caviar Protocol | 7th |    [link](https://code4rena.com/contests/2023-04-caviar-private-pools) |
| Reserve Protocol | 9th |    [link](https://code4rena.com/contests/2023-01-reserve-contest) |

All my public bug reports can be found in [public-audits](public-audits/README.md).


### Interesting bugs that I have found

- First deposit bug in Ondo Finance (fork of Compound V2).

    The report shows how a token balance inflation attack can be performed on the protocol to steal user's deposit. More details in my blog post [here](https://www.akshaysrivastav.com/articles/first-deposit-bug-in-compound-v2) and in the [report](/public-audits/2023-01-ondo/[MEDIUM]-First_deposit_bug/README.md).

- Broken fallback price mechanism in Gravita Protocol

    The report demonstrate the broken fallback price oracle implementation of the protocol which can lead to protocol suffering a complete DoS. More details in the [report](/public-audits/2023-04-gravita/[MEDIUM]-PriceFeed:_Incorrectfallback_price_mechanism_leading_to_protocol_DoS/README.md).

- Incorrect implementation of cross-chain smart contract system in PoolTogether protocol.
    
    This report shows how an incorrect implementation of cross chain system can cause loss of funds to the connecting transport layer. More details in the [report](/public-audits/2022-12-pooltogether/[MEDIUM]-'CrossChainExecutor'_contracts_do_not_update_the_necessary_states_for_failing_transactions./README.md).

- Critical monetary loss bug in GoGoPool (an Ethereum staking protocol).
    
    This report shows how the funds staked by users in the staking protocol can be nullified by an attacker causing loss of funds to users. More details in the [report](/public-audits/2022-12-gogopool/[HIGH]-Funds_of_Node_Operators_can_be_nullified_by_any_attacker/README.md).

- Frontrunning the use of CREATE2 in Caviar protocol.
    
    This report demonstrates how the inefficient use of CREATE2 can be exploited by front-running to steal user's funds. More details in the [report](/public-audits/2023-04-caviar/[MEDIUM]-'Factory.create':_Predictability_of_pool_address_creates_multiple_issues./README.md).

### Some of my High severity findings

| Audit Contest   |      Finding      |  Details |
|----------|:-------------|:------:|
| Caviar Protocol |  Funds can be stolen from pool due to inefficient royalty distribution | [link](/public-audits/2023-04-caviar/[HIGH]-Funds_can_be_stolen_from_pool_due_to_inefficient_royalty_distribution/README.md) |
| Rabbithole Protocol |  `withdrawRemainingTokens` and `withdrawFee` functions can be used to pull out user funds | [link](/public-audits/2023-01-rabbithole/[HIGH]-'withdrawRemainingTokens'_and_'withdrawFee'_functions_can_be_used_to_pull_out_user_funds/README.md) |
| GoGoPool Protocol |  Funds of Node Operators can be nullified by any attacker | [link](/public-audits/2022-12-gogopool/[HIGH]-Funds_of_Node_Operators_can_be_nullified_by_any_attacker/README.md) |
| Escher Protocol |  Loss of ETH for NFT buyers  | [link](/public-audits/2022-12-escher/[HIGH]-Loss_of_ETH_for_NFT_buyers_in_LPDA_contract/README.md) |


Beyond these reports, some of my findings has been kept private on protocol's requests. Results of some public audit contests and bounties are still pending, I'll add those once they are announced.

### Private Audits
All my private audit contributions can be found in [private-audits](private-audits/README.md).

