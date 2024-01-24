### [M-04] Risk of Inliquidable Vaults due to ERC20 Tokens with Blacklist Functionality

**Description**

Certain ERC20 tokens, like USDC, possess the capability to blacklist specific addresses, effectively impeding their ability to send or receive tokens. Transactions involving these blacklisted addresses result in reverts. Integrating such tokens in the future might result in smart vaults that cannot be liquidated. The vulnerability is situated within [`LiquidationPoolManager::forwardRemainingRewards()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L43). This function utilizes `.transfer()` exclusively instead of the safer `.safeTransfer()` method, observed specifically on [line 54](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L54). Furthermore, it lacks the provision to specify a `to` recipient address. This situation could lead to significant issues when attempting to transfer funds to the protocol's multisig, especially if this address is blacklisted by any token within the `_tokens` array.
```javascript
function forwardRemainingRewards(ITokenManager.Token[] memory _tokens) private {
    for (uint256 i = 0; i < _tokens.length; i++) {
        ITokenManager.Token memory _token = _tokens[i];
        if (_token.addr == address(0)) {
            uint256 balance = address(this).balance;
            if (balance > 0) {
                (bool _sent,) = protocol.call{value: balance}("");
                require(_sent);
            }
        } else {
            uint256 balance = IERC20(_token.addr).balanceOf(address(this));
@>          if (balance > 0) IERC20(_token.addr).transfer(protocol, balance); // @audit possible revert if "protocol" is blacklisted
        }
    }
}
```

**Impact**

Incorporating ERC20 tokens with blacklist functionality into the accepted collateral tokens might result in the protocol multisig being unable to claim the remaining rewards following a smart vault's liquidation. This would cause the liquidation to revert, thereby disrupting a critical functionality with a high impact.

**Recommended Mitigation**

To mitigate this issue, consider modifying the function to include a `to` address variable within the function parameters. This enhancement would enable the specification of the recipient address for transferred funds, ensuring a more secure transfer process.
```diff
- function forwardRemainingRewards(ITokenManager.Token[] memory _tokens) private {
+ function forwardRemainingRewards(ITokenManager.Token[] memory _tokens, address to) private {
    for (uint256 i = 0; i < _tokens.length; i++) {
        ITokenManager.Token memory _token = _tokens[i];
        if (_token.addr == address(0)) {
            uint256 balance = address(this).balance;
            if (balance > 0) {
-               (bool _sent,) = protocol.call{value: balance}("");
+               (bool _sent,) = to.call{value: balance}(""); // this change is not mandatory, because it won't revert
                require(_sent);
            }
        } else {
            uint256 balance = IERC20(_token.addr).balanceOf(address(this));
-           if (balance > 0) IERC20(_token.addr).transfer(protocol, balance);
+           if (balance > 0) IERC20(_token.addr).transfer(to, balance);
        }
    }
}
```

**Tools Used**

Manual Review