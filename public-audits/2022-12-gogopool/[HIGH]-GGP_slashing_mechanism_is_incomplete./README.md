# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/845
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/Staking.sol#L379-L383


# Vulnerability details

## Impact
The protocol [docs](https://multisiglabs.notion.site/Architecture-Protocol-Overview-4b79e351133f4d959a65a15478ec0121) mentions that "If the validator is failing at their duties, their GGP will be slashed and used to compensate the loss to our Liquid Stakers."

But the actual implementation of the `Staking.slashGGP` function is very different from the above specification. 
```solidity
	function slashGGP(address stakerAddr, uint256 ggpAmt) public onlySpecificRegisteredContract("MinipoolManager", msg.sender) {
		Vault vault = Vault(getContractAddress("Vault"));
		decreaseGGPStake(stakerAddr, ggpAmt);
		vault.transferToken("ProtocolDAO", ggp, ggpAmt);
	}
```
The few issues in the above code can be listed as:
 - The contracts do not distribute the slashed amount to Liquid stakers or ggAVAX holders (unlike as stated in the docs).
 - The contract simply transfer the GGP token amount to ProtocolDAO contract via Vault. Currently ProtocolDAO contract do not contain any function to utilize or recover those transferred GGP tokens. To utilize the GGP funds the ProtocolDAO contract will be needed to be upgraded.

## Proof of Concept
Consider this scenario:

1. A node operator do not maintains sufficient uptime and hence the multisig decides to slash his GGP stake.
2. The multisig invokes `MinipoolManager.recordStakingEnd` function which invokes the `Staking.slashGGP` function.
3. The slashed rewards get sent to ProtocolDAO contract via Vault. This GGP balance of ProtocolDAO contract in Vault simply sits idle and unusable until the ProtocolDAO contract is upgraded with logic to utilize that GGP balance.

## Tools Used
Manual review

## Recommended Mitigation Steps
The sponsors should re-implelment the `Staking.slashGGP` function with alignment to the protocol docs so that the contract natively distribute the slashed GGP to liquid stakers.
Or,
The ProtocolDAO contract should be modified so that it can natively handle the transferred GGP token balance.