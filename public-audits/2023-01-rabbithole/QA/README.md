# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/179
1. Some upgradable contract implementations use `_disableInitializers` and some use `initializer`. These different uses have different outputs.
    While `initializer` modifier sets the `_initialized` variable to `1`, the `_disableInitializers` sets `_initialized` to `type(uint8).max`. A single way of disabling implementation contracts must be used for all contracts to maintain consistensy.
2. `RabbitHoleTickets.setTicketRenderer` and `setRoyaltyRecipient` should emit events.
    ```solidity
    function setTicketRenderer(address ticketRenderer_) public onlyOwner {
        TicketRendererContract = TicketRenderer(ticketRenderer_);
    }
    function setRoyaltyRecipient(address royaltyRecipient_) public onlyOwner {
        royaltyRecipient = royaltyRecipient_;
    }
    ```
3. Name of unused variables can be omitted to silence compiler warnings.
    ```solidity
    function royaltyInfo(
        uint256 tokenId_,
        uint256 salePrice_
    ) external view override returns (address receiver, uint256 royaltyAmount) {
        // ...
    }
    ```
    Can be changed to 
    ```solidity
    function royaltyInfo(
        uint256,
        uint256 salePrice_
    ) external view override returns (address receiver, uint256 royaltyAmount) {
        // ...
    }
    ```
4. In `Quest.claim` the updation of `redeemedTokens` breaks CEI pattern.
    ```solidity
    function claim() public virtual onlyQuestActive {
        // ...

        uint256 totalRedeemableRewards = _calculateRewards(redeemableTokenCount);
        _setClaimed(tokens);       
        _transferRewards(totalRedeemableRewards);
        redeemedTokens += redeemableTokenCount;     // breaking CEI - state updated after external call

        // ...
    }
    ```
5. Quest - compiler warnings for unreachable code can be silenced by marking the Quest contract as abstract with declared functions rather than reverting.
    ```solidity
    abstract contract Quest ... {
        function _calculateRewards(uint256 redeemableTokenCount_) internal virtual returns (uint256);
    
        function _transferRewards(uint256 amount_) internal virtual;
    }
    ```