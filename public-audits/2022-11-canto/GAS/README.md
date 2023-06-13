# Original link
https://github.com/code-423n4/2022-11-canto-findings/issues/42
1. In the `withdraw` function, due to Line 135 `_amount` value can never exceed `earnedFees`, the Line 137 can be put inside an `unchecked` block.
```solidity
135        if (_amount > earnedFees) _amount = earnedFees;
137
137        balances[_tokenId] = earnedFees - _amount;
```

2. At L50, L87 and L108 the `msg.sender` value is stored in an `address` variable in memory. This value is then read and used at multiple places. It is recommended to use `msg.sender` directly without storing/reading from memory to save gas. 
```solidity
    function register(address _recipient) public onlyUnregistered returns (uint256 tokenId) {
        address smartContract = msg.sender;

        // code clipped...

        emit Register(smartContract, _recipient, tokenId);

        feeRecipient[smartContract] = NftData({
            tokenId: tokenId,
            registered: true
        });
    }
```

3. It is recommended to simply use a `uint256` variable instead if `Counters` library to save gas.

4. <x> += <y> costs more gas than <x> = <x> + <y> for state variables. 
```solidity
        balances[_tokenId] += msg.value;
```