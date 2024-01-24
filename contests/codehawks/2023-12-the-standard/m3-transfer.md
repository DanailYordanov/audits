### [M-03] Risk of Locked Rewards due to ERC20 Tokens with Blacklist Functionality

**Description**

Certain ERC20 tokens, like USDC, feature the capability to blacklist specific addresses, effectively restricting their ability to send or receive tokens. Transactions involving these blacklisted addresses result in reverts. The utilization of such tokens in the future could lead to funds becoming inaccessible. The vulnerability is situated within [`LiquidationPool::claimRewards()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L164). This function solely employs `.transfer()` instead of the safer `.safeTransfer()` on [line 175](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L175) and lacks the provision to specify a `to` address. Consequently, should a holder's account be blacklisted by one of their reward tokens, invoking the claim function consistently results in a revert.
```javascript
function claimRewards() external {
    ITokenManager.Token[] memory _tokens = ITokenManager(tokenManager).getAcceptedTokens();
    for (uint256 i = 0; i < _tokens.length; i++) {
        ITokenManager.Token memory _token = _tokens[i];
        uint256 _rewardAmount = rewards[abi.encodePacked(msg.sender, _token.symbol)];
        if (_rewardAmount > 0) {
            delete rewards[abi.encodePacked(msg.sender, _token.symbol)];
            if (_token.addr == address(0)) {
                (bool _sent,) = payable(msg.sender).call{value: _rewardAmount}("");
                require(_sent);
            } else {
@>              IERC20(_token.addr).transfer(msg.sender, _rewardAmount); // @audit would revert on blacklist, the transfer should be to "to" instead of "msg.sender" and preferably with .safeTransfer() 
            }   
        }

    }
}
```
**Impact**

Incorporating ERC20 tokens featuring blacklist functionality into the accepted collateral tokens could render a holder incapable of claiming their complete rewards from liquidated smart vaults if their address gets blacklisted by one of the reward tokens.

**Recommended Mitigation**

To address this issue, consider implementing a `to` address variable within the function, providing the option to designate the recipient address for transferred funds.
```diff
- function claimRewards() external {
+ function claimRewards(address to) external {
    ITokenManager.Token[] memory _tokens = ITokenManager(tokenManager).getAcceptedTokens();
    for (uint256 i = 0; i < _tokens.length; i++) {
        ITokenManager.Token memory _token = _tokens[i];
        uint256 _rewardAmount = rewards[abi.encodePacked(msg.sender, _token.symbol)];
        if (_rewardAmount > 0) {
            delete rewards[abi.encodePacked(msg.sender, _token.symbol)];
            if (_token.addr == address(0)) {
                (bool _sent,) = payable(msg.sender).call{value: _rewardAmount}("");
                require(_sent);
            } else {
-               IERC20(_token.addr).safeTransfer(msg.sender, _rewardAmount);
+               IERC20(_token.addr).safeTransfer(to, _rewardAmount);
            }   
        }

    }
}
```

**Tools Used**

Manual Review