# Original link
https://github.com/Gravita-Protocol/Gravita-SmartContracts/issues/233



# Vulnerability details

`__Ownable_init` function is not invoked in `StabilityPool` and `VesselManager` contracts.



## Affected smart contract

- StabilityPool
- VesselManager


## Description
The `StabilityPool` and `VesselManager` contracts inherits `GravitaBase` which itself inherits `OwnableUpgradeable`. Hence these contracts contains an `_owner` state variable. This variable should ideally by initialised using the `OwnableUpgradeable.__Ownable_init` function. But that initialization is missing in these contracts.

Due to this the `StabilityPool.owner()` and `VesselManager.owner()` function returns `0x00` address even after the contracts are initialised. There is no way for these contracts to set the `_owner` state variable ever.

The smart contract API is scanned and queried by multiple offchain agents (like etherscan, graph protocol, etc). The presence of the `owner()` function which returns `0x00` value incorrectly can cause integration issues with Gravitas contracts.

Moreover, the `GravitaBase` is also inherited by `BorrowerOperations` and `VesselManagerOperations` contracts and the `__Ownable_init` function is invoked correctly in them. The uninitialised state of `_owner` breaks the inhouse smart contract design of Gravitas contracts.

## Attack scenario
Poc:

```solidity
const { ethers } = require("hardhat");
const StabilityPool = artifacts.require("StabilityPool")
const VesselManager = artifacts.require("VesselManager")

contract("StabilityPool", async () => {

    it("Uninitialised owner", async () => {
        const AddressZero = ethers.constants.AddressZero
        const AddressOne = "0x0000000000000000000000000000000000000001"

        const stabilityPool = await StabilityPool.new()
        const vesselManager = await VesselManager.new()

        expect(await stabilityPool.owner()).to.eq(AddressZero)
        expect(await vesselManager.owner()).to.eq(AddressZero)

        // setting dummy addresses
        await stabilityPool.setAddresses(
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne
        )
        await vesselManager.setAddresses(
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne,
            AddressOne
        )

        expect(await stabilityPool.owner()).to.eq(AddressZero)
        expect(await vesselManager.owner()).to.eq(AddressZero)
    })
})
```

## Recommendation

Consider adding `__Ownable_init` statement in the `setAddresses` function of `StabilityPool` and `VesselManager` contracts

```solidity
	function setAddresses(
		...
	) external initializer {
		__Ownable_init();
		...
	}
```
