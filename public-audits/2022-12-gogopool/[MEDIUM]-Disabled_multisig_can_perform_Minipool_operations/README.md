# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/538
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/Ocyticus.sol#L55
https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L57


# Vulnerability details

## Impact
In GoGoPool protocol, all multisigs can be disabled using the `Ocyticus.pauseEverything` and `Ocyticus.disableAllMultisigs` functions. It should also be noted that in `MinipoolManager` every Minipool gets assigned a mulstisig at the time of that Minipool creation.

After a multisig has been disabled it can still perform various operations on the Minipool for which it was assigned as a valid multisig. The operations includes:
 - claimAndInitiateStaking
 - recordStakingStart
 - recordStakingEnd
 - recreateMinipool (when not paused)
 - recordStakingError
 - cancelMinipoolByMultisig
 - finishFailedMinipoolByMultisig 

The disabling of multisigs can be done for various reasons (including private key compromises) and letting disabled multisigs perform crucial operations on Minipools is not ideal.

## Proof of Concept
Consider this scenario:

 - A multisig M1 was assigned to a Minipool.
 - Then `Ocyticus.disableAllMultisigs` was invoked and all multisigs were disabled.
 - Still the multisig M1 can invoke various functions for the Minipool (claimAndInitiateStaking, recordStakingEnd, recreateMinipool, recordStakingError, cancelMinipoolByMultisig, finishFailedMinipoolByMultisig) which can cause unintended loss of funds to the users or can result in ambiguous state for the protocol contracts. 

## Tools Used
Manual review

## Recommended Mitigation Steps
The protocol should check the current `enabled` or `disabled` state of the caller multisigs before allowing it to perform any operation on the Minipool. The protocol should also have a way to upgrade the assigned multisig for a Minipool.