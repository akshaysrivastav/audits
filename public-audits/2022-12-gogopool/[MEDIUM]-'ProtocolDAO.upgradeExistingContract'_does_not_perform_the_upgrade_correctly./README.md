# Original link
https://github.com/code-423n4/2022-12-gogopool-findings/issues/542
# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/ProtocolDAO.sol#L209-L216


# Vulnerability details

## Impact
The `ProtocolDAO.upgradeExistingContract` is intended to register a new contract in protocol and unregister the old contract. It essentially combined `registerContract` and `unregisterContract` function calls in a single call.

```solidity
	function upgradeExistingContract(
		address newAddr,
		string memory newName,
		address existingAddr
	) external onlyGuardian {
		registerContract(newAddr, newName);
		unregisterContract(existingAddr);
	}
```

According to its implementation it can be seen that it first registers the new contract and then unregisters the old one. This sequence causes issues if the new and old name of the contract is same. In that case the storage values gets messed up.

## Proof of Concept

```solidity
contract BugTest is Test {
	Storage public store;
	ProtocolDAO public dao;

	function setUp() public {
		store = new Storage();
		dao = new ProtocolDAO(store);
		store.setBool(keccak256(abi.encodePacked("contract.exists", address(dao))), true);
        }

        function test_upgradeExistingContract() public {
        	string memory contractName = "Oracle";
        	address existingAddr = address(0x100);

        	dao.registerContract(existingAddr, contractName);

        	address newAddr = address(0x200);
        	dao.upgradeExistingContract(newAddr, contractName, existingAddr);

        	assertEq(store.getBool(keccak256(abi.encodePacked("contract.exists", existingAddr))), false);
		assertEq(store.getAddress(keccak256(abi.encodePacked("contract.address", contractName))), address(0));
		assertEq(store.getString(keccak256(abi.encodePacked("contract.name", existingAddr))), "");
        	assertEq(store.getBool(keccak256(abi.encodePacked("contract.exists", newAddr))), true);
		assertEq(store.getAddress(keccak256(abi.encodePacked("contract.address", contractName))), address(0));
		assertEq(store.getString(keccak256(abi.encodePacked("contract.name", newAddr))), "Oracle");
    	}
}

```
In the test case above, the `newAddr` address value points to "Oracle" string but the "Oracle" string points to null address.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider performing unregistering the old contract before registering the new one or consider validating that new and old contract names cannot be same.