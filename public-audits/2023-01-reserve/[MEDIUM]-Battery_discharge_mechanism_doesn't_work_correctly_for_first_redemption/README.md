# Original link
https://github.com/code-423n4/2023-01-reserve-findings/issues/452
# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/libraries/RedemptionBattery.sol#L59-L70


# Vulnerability details

## Impact
The `RTokenP1` contract implements a throttling mechanism using the `RedemptionBatteryLib` library. The library models a "battery" which "recharges" linearly block by block, over roughly 1 hour.

RToken.sol
```solidity
    function redeem(uint256 amount) external notFrozen {
        // ...

        uint256 supply = totalSupply();

        // ...
        battery.discharge(supply, amount); // reverts on over-redemption

        // ...
    }
```
RedemptionBatteryLib.sol
```solidity
    function discharge(
        Battery storage battery,
        uint256 supply,
        uint256 amount
    ) internal {
        if (battery.redemptionRateFloor == 0 && battery.scalingRedemptionRate == 0) return;

        // {qRTok}
        uint256 charge = currentCharge(battery, supply);

        // A nice error message so people aren't confused why redemption failed
        require(amount <= charge, "redemption battery insufficient");

        // Update battery
        battery.lastBlock = uint48(block.number);
        battery.lastCharge = charge - amount;
    }

    /// @param supply {qRTok} Total RToken supply before the burn step
    /// @return charge {qRTok} The current total charge as an amount of RToken
    function currentCharge(Battery storage battery, uint256 supply)
        internal
        view
        returns (uint256 charge)
    {
        // {qRTok/hour} = {qRTok} * D18{1/hour} / D18
        uint256 amtPerHour = (supply * battery.scalingRedemptionRate) / FIX_ONE_256;

        if (battery.redemptionRateFloor > amtPerHour) amtPerHour = battery.redemptionRateFloor;

        // {blocks}
        uint48 blocks = uint48(block.number) - battery.lastBlock; 

        // {qRTok} = {qRTok} + {qRTok/hour} * {blocks} / {blocks/hour}
        charge = battery.lastCharge + (amtPerHour * blocks) / BLOCKS_PER_HOUR;

        uint256 maxCharge = amtPerHour > supply ? supply : amtPerHour;
        if (charge > maxCharge) charge = maxCharge;
    }
```
The linear redemption limit is calculated in the `currentCharge` function. This function calculates the delta blocks by `uint48 blocks = uint48(block.number) - battery.lastBlock;`.

The bug here is that the `lastBlock` value is never initialized by the `RTokenP1` contract so its value defaults to `0`. This results in incorrect delta blocks value as the delta blocks comes out to be an incorrectly large value
```
        blocks = current block number - 0 = current block number
```

Due do this issue, the `currentCharge` value comes out to be way larger than the actual intended value for the first RToken redemption. The `maxCharge` cap at the end of `currentCharge` function caps the result to the current total supply of RToken. 

The issue results in an instant first RToken redemption for the full `totalSupply` of the RToken. The battery discharging mechanism is completely neglected.

It should be noted that the issue only exists for the first ever redemption as during the first redemption the `lastBlock` value gets updated with current block number.  


## Proof of Concept
The following test case was added to `test/RToken.test.ts` file and was ran using command `PROTO_IMPL=1 npx hardhat test ./test/RToken.test.ts`.

```typescript
  describe.only('Battery lastBlock bug', () => {
    it('redemption battery does not work on first redemption', async () => {
      // real chain scenario
      await advanceBlocks(1_000_000)
      await Promise.all(tokens.map((t) => t.connect(addr1).approve(rToken.address, ethers.constants.MaxUint256)))

      expect(await rToken.totalSupply()).to.eq(0)
      await rToken.connect(owner).setRedemptionRateFloor(fp('1e4'))
      await rToken.connect(owner).setScalingRedemptionRate(fp('0'))

      // first issue
      const issueAmount = fp('10000')
      await rToken.connect(addr1)['issue(uint256)'](issueAmount)
      expect(await rToken.balanceOf(addr1.address)).to.eq(issueAmount)
      expect(await rToken.totalSupply()).to.eq(issueAmount)

      // first redemption
      expect(await rToken.redemptionLimit()).to.eq(await rToken.totalSupply())    // for first redemption the currentCharge value is capped by rToken.totalSupply() 
      await rToken.connect(addr1).redeem(issueAmount)
      expect(await rToken.totalSupply()).to.eq(0)

      // second redemption
      await rToken.connect(addr1)['issue(uint256)'](issueAmount)
      expect(await rToken.balanceOf(addr1.address)).to.eq(issueAmount)
      // from second redemtion onwards the battery discharge mechanism takes place correctly
      await expect(rToken.connect(addr1).redeem(issueAmount)).to.be.revertedWith('redemption battery insufficient')
    })
  })
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
The `battery.lastBlock` value must be initialized in the `init` function of `RTokenP1`
```solidity
    function init(
        // ...
    ) external initializer {
        // ...
        battery.lastBlock = uint48(block.number);
    }
``` 