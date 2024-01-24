### [M-08] Inaccurate Swap Fee Calculation May Cause Swap Reversion

**Description**

The vulnerability exists within the [`SmartVaultV3::swap()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L214) function. The issue arises when calculating the `minimumAmountOut` parameter for `ISwapRouter.ExactInputSingleParams` in the [`SmartVaultV3::calculateMinimumAmountOut()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L206) function. This parameter is crucial, especially in risky swaps, as it influences whether the Smart Vault might become liquidatable. The problem lies in not factoring the [`swapFee`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L215) into the `calculateMinimumAmountOut()` call, whereas it's subtracted from the [`amountIn`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L224) parameter in the `ISwapRouter.ExactInputSingleParams` configuration.

```javascript
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
    uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    address inToken = getSwapAddressFor(_inToken);
@>  uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount); // @audit swapFee not included
    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: inToken,
            tokenOut: getSwapAddressFor(_outToken),
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: _amount - swapFee, // @audit swapFee deducted here
            amountOutMinimum: minimumAmountOut, // @audit minimumAmountOut might be higher than amountIn in some cases
            sqrtPriceLimitX96: 0
        });
    inToken == ISmartVaultManagerV3(manager).weth() ?
        executeNativeSwapAndFee(params, swapFee) :
        executeERC20SwapAndFee(params, swapFee);
}
```

**Impact**

Failure to include the `swapFee` when calculating the `minimumAmountOut` could result in a swap reversion or lead to an unpleasant user experience, especially in highly volatile market conditions.

**Proof of Concept**

Consider the following scenario:

1. Bob has 1600 USDC and utilizes the entirety as collateral, reaching the maximum mintable amount.
2. Bob attempts to swap his USDC for 1 ETH using [`SmartVaultV3::swap()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L214).
3. The calculated [`swapFee`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L215) is 50 USDC.
4. The calculated [`minimumAmountOut`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L217) is 1600 USDC.
5. The inputted [`amountIn`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L224) becomes 1600 - 50 = 1550 USDC.
6. Due to the higher `minimumAmountOut` value than `amountIn`, the swap reverts.

**Recommended Mitigation**

Include the `swapFee` in the calculation for [`minimumAmountOut`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L217) as follows:

```diff
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
    uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    address inToken = getSwapAddressFor(_inToken);
-   uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
+   uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount - swapFee);
    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: inToken,
            tokenOut: getSwapAddressFor(_outToken),
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: _amount - swapFee,
            amountOutMinimum: minimumAmountOut,
            sqrtPriceLimitX96: 0
        });
    inToken == ISmartVaultManagerV3(manager).weth() ?
        executeNativeSwapAndFee(params, swapFee) :
        executeERC20SwapAndFee(params, swapFee);
}
```

**Tools Used**

Manual Review
