# ZKsync Governance System Threat Modeling Exercise

## Participant Information 
Name: Akshay Srivastav

Role: Security Researcher

Organization: Spearbit

Contact Information: mail@akshaysrivastav.com

Date of Review: August 12th, 2024

### Summary
8 impactful vulnerabilities, 3 threat vectors, 7 low/informational issues & 7 documents related issues were found.

## Vulnerabilities
##### 1. Emergency upgrades can be replayed infinite times on L1.
###### Impact: High
###### Likelihood: High
###### Description: 
The `EmergencyUpgradeBoard.executeEmergencyUpgrade` lacks signature replay protection. So an emergency upgrade can be executed repeatedly by passing the same signatures again. This can lead to ambiguous onchain state for ZKsync protocol and can also lead to significant financial losses to users.

The `ProtocolUpgradeHandler.executeEmergencyUpgrade` also doesn't prevent replaying of upgrade proposals. Even after an emergency upgrade proposal has been executed, the `upgradeState` function still returns `UpgradeState.None` as the state of that emergency upgrade proposal. Hence replay becomes possible.

###### Mitigation: 
Implement signature replay protection using `nonces` in `EmergencyUpgradeBoard.executeEmergencyUpgrade`. Also consider setting `creationTimestamp` of the emergency upgrade proposal in `ProtocolUpgradeHandler.executeEmergencyUpgrade`.

##### 2. Protocol can get unfrozen by a parallel upgrade.
###### Impact: High
###### Likelihood: Low
###### Description:
Consider this scenario:
1. A minor upgrade proposal `P` was approved on L2 and L1. The upgrade proposal is in `ExecutionPending` state on L1.
2. The SecurityCouncil notices a bug on ZKsync and hard freezes the ZKsync protocol.
3. The waiting delay of proposal `P` has now passed and the proposal is now `Ready`.
4. Any user can now execute the proposal `P` which will also unfreeze the entire ZKsync protocol. This may expose ZKsync to the bug found in step 2. 

These kind of parallel proposals can unintentionally unfreeze the frozen ZKsync protocol.

##### 3. Signatures of governance bodies do not expire.
###### Impact: Low
###### Likelihood: High
###### Description:
The signatures provided by the members of Security Council and Guardian multisigs for these functions never expire:
- `Guardian.extendLegalVeto`
- `Guardian.approveUpgradeGuardians`
- `Guardian.proposeL2GovernorProposal`
- `Guardian.cancelL2GovernorProposal`
- `SecurityCouncil.approveUpgradeSecurityCouncil`

Any unused signature generated for these functions can be used anytime in the future (assuming that the on-chain operation wasn't executed).

###### Mitigation:
Consider also including an `expiry` parameter while generating the signature digest for the above mentioned function. It should be validated that `expiry > block.timestamp`. 

##### 4. Approved upgrades can be executed anytime into the future.
###### Impact: High
###### Likelihood: Medium
###### Description:
In `ProtocolUpgradeHandler` once a proposal has been approved by Security Council or Guardians it can then be executed anytime into the future. There is no expiry of an approved upgrade.
In case a proposal cannot be executed once it is ready, due to execution getting reverted, then this proposal will remain in ready state forever. Hence it can be executed anytime into the future which could be unintentional for ZKsync protocol. 

###### Mitigation:
Consider implementing an expiry for the approved upgrade proposals.

##### 5. `ProtocolUpgradeHandler.reinforceUnfreeze` can be called arbitrarily in healthy protocol state.
###### Impact: Medium
###### Likelihood: High
###### Description:
The `reinforceUnfreeze` and `reinforceUnfreezeOneChain` functions can be called arbitrarily by any user anytime, even when the protocol isn't even frozen in the first place. This function performs the unfreeze operations on all hyperchains and bridges. Hence any unintentional call can lead to ambiguous state for the protocol.

##### 6. Unbounded `for` loop execution can revert due to block gas limit.
###### Impact: High
###### Likelihood: Low
###### Description:
in `ProtocolUpgradeHandler` the freeze operation tries to freeze all existing hyperchains in a `for` loop, similar is the case for unfreeze. In case the number of existing hyperchains grows to a bigger value then this `for` loop execution can get reverted due to ethereum's block gas limit. In case this ever happens then freezing or unfreezing of ZKsync protocol won't be possible in emergency scenarios. 

##### 7. In `Multisig.checkSignatures` function `0` value can be provided as `_threshold` parameter.
###### Impact: Low
###### Likelihood: Low
###### Description:
The `checkSignatures` accepts `0` as `_threshold` parameter in which case the `for` loop and the `isValidSignatureNow` check doesn't get executed.
###### Mitigation:
Consider validating that the `_threshold` parameter is not `0`.

##### 8. `ProtocolUpgradeHandler`: Missing `_chainId` validation for reinforce functions
###### Impact: Low
###### Likelihood: High
###### Description:
The `reinforceFreezeOneChain` and `reinforceUnfreezeOneChain` are publicly accessible functions which accepts a `_chainId` parameter from user. These functions then call `freezeChain` and `unfreezeChain` functions on `STATE_TRANSITION_MANAGER`. Any arbitrary `_chainId` value may result in unintended and ambigiuos states for ZKsync protocol contracts.

###### Mitigation:
Consider validating the `_chainId` parameter. 

## Threats

##### 1. If Guardian on ZKsync goes rogue then no proposals can be passed
###### Impact: Medium
###### Likelihood: Low
###### Description:
The `ZkGovOpsGovernor` and `ZkTokenGovernor` contracts give an exclusive right to `VETO_GUARDIAN` to cancel any proposals. In case the `VETO_GUARDIAN` entity gets compromised or goes rogue then `VETO_GUARDIAN` will have the ability to cancel all existing and future proposals of the governors. Hence no proposals can be passed.
To mitigate this issue Token Assembly will need to pass a proposal to upgrade the impacted `ZkGovOpsGovernor` or `ZkTokenGovernor` contract via `ZkProtocolGovernor`. But any contract which is directly governed by these impacted governors may still be in full control of them.

##### 2. The `ZkProtocolGovernor` is fully controlled by ZKsync token holders
###### Impact: Medium
###### Likelihood: Medium
###### Description:
Unlike other governor contracts the `ZkProtocolGovernor` is not protected by `VETO_GUARDIAN`. It is fully in control of ZKsync token holders.
It is possible for a malicious entity to gain significant voting rights on a token based governance protocol, by either purchasing the token from market or by bribing the existing token holders (eg, [Compound Finance](https://www.comp.xyz/t/trust-setup-for-dao-investment-into-goldcomp/5406/3)). In case any smart contract is directly managed by `ZkProtocolGovernor` then token holders will have full control over that smart contract's management.

##### 3. Collaboration overhead for Security Council members
###### Impact: Low
###### Likelihood: Medium
###### Description:
The `softFreeze`, `hardFreeze`, `unfreeze` & `setSoftFreezeThreshold` functions of `SecurityCouncil` requires a `_validUntil` timestamp value as input parameter. This value is the expiry timestamp of member signatures. Since a single `_validUntil` value is used to check the validity of all member signatures, all members will need to collaborate before generating signatures and agree upon a common `_validUntil` value which suites all of them. This can cause an additional overhead during emergency situations. 
###### Mitigation:
Individual expiry values can be used for every member signature. 

## Low/Informational Issues
1. `ProtocolUpgradeHandler.approveUpgradeSecurityCouncil` should explicitly prevent double approval by security council. Consider validating that `securityCouncilApprovalTimestamp` for the proposal in `0`.
2. Incorrect comment for `SecurityCouncil.softFreeze`. The natspec of `_validUntil` mentions that it is "The timestamp until which the signature should remain valid", however the function reverts when `block.timestamp == _validUntil`.
3. In `Multisig.constructor` consider adding a check to validate that `currentMember != address(this)`.
4. Remove the `Counter` contract from `l2-contracts/src`.
5. In `SecurityCouncil.setSoftFreezeThreshold` replace `SOFT_FREEZE_CONSERVATIVE_THRESHOLD` with `EIP1271_THRESHOLD` during the `checkSignatures` call.
6. `SecurityCouncil.unfreeze` is missing resetting `softFreezeThreshold` to `RECOMMENDED_SOFT_FREEZE_THRESHOLD` after unfreezing.
7. Any governance contract which deals with signatures must handle signature validations securely. No signature should be replayable once used and all signatures must expire after a time period. Consider using `nonces` and `expiry` parameters for all signature validations. 

## Issues in non-smart contract documents
1. [ZKsync Proposal lifecycle diagram](https://drive.google.com/file/d/16ek76cniH3xmkfNzE8G2Y93VSjS6hTM6/view)
    - In `Protocol Governor Proposal Process :: Offchain Veto Period` - change `voted` -> `vetoed`.
    - The `Voting Period` section says that "Voting is extended to 7 more days in case a quorum of 3% of
total supply is reached within 24 hours". But as per `GovernorPreventLateQuorum` contract voting is always extended whenever quorum is reached irrespective of point in time when quorum is reached.
    - Protocol Governor responsibilities section is missing "Upgrades to Emergency Upgrade Board" responsibility

2. [Governance design diagram](https://www.figma.com/board/tI16cuEVFVjTMQjPn5IOwD/ZKsync-Governance-Contracts)
    - Protocol Governor flow says that 9/12 SC members must approve a proposal but on SecurityCouncil smart contract 6/12 members can approve a proposal.
    - Emergency Response flow says that specified party performs the emergency upgrade execution but as per `EmergencyUpgradeBoard` contract anyone can collect signatures and perform the execution.
    - The Token and GovOps Governor flow says that the quorum is `FOR / FOR + AGAINST` votes, but actually on smart contracts quorum is `FOR > Admin_Controlled_Threshold`.

3. [Governance Procedure](https://docs.google.com/document/d/1wx_fXSGonjvVVigY7WsnUNCDMblqEalN)
    - Schedule 2 - Point 6.4 says that, "An Emergency Upgrade may be initiated by any individual Signer of the Security Multisig, any individual Signer of the Guardian Multisig, or the ZKsync Foundation Multisig", however the emergency upgrade can be initiated by any user. The signatures can be collected from pending upgrade transaction and then be submitted in a front-running fashion. 
