### [M-07] Lack of Slippage Deadline Protection in Swaps Leads to Token Selling at Inaccurate Prices

**Description**

Within [`SmartVaultV3::swap()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L214), the `deadline` parameter is set as `block.timestamp` on [line 223](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L223). This configuration disables the transaction expiration check, rendering the deadline ineffective. Uniswap, for instance, utilizes the deadline argument to safeguard users from executing transactions at outdated prices, particularly prices lower than the current market rate.

```javascript
function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
    uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    address inToken = getSwapAddressFor(_inToken);
    uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
    ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: inToken,
            tokenOut: getSwapAddressFor(_outToken),
            fee: 3000,
            recipient: address(this),
@>          deadline: block.timestamp, // @audit no protection
            amountIn: _amount - swapFee,
            amountOutMinimum: minimumAmountOut,
            sqrtPriceLimitX96: 0
        });
    inToken == ISmartVaultManagerV3(manager).weth() ?
        executeNativeSwapAndFee(params, swapFee) :
        executeERC20SwapAndFee(params, swapFee);
}
```

**Impact**

The absence of slippage protection in the form of a transaction deadline allows for the possibility of tokens being sold at a price lower than the current market rate during a swap. This vulnerability opens the door to exploitation, particularly through a sandwich attack.

**Proof of Concept**

Consider the following scenario:

1. Bob initiates a swap for 1400 USDC for 1 ETH.
2. Before the transaction is mined, a rapid increase in gas costs occurs, causing the transaction to linger in the mempool due to its lower gas price compared to the current rate.
3. While the transaction remains in the mempool, the price of ETH surges.
4. Once the gas cost drops and the transaction is mined, the value of `amountOutMinimum` calculated at the outdated rate allows a sandwich attack by a MEV bot. The bot manipulates the Uniswap pool, lowering the price of the reward token to fulfill the minimum output amount check, profiting from the swap occurring at a reduced price.
5. Resultantly, the reward tokens are swapped at an outdated and lower price, causing Bob to earn less yield than anticipated if the tokens were sold at the current market price.

**Recommended Mitigation**

Implement a reasonable deadline value for the argument, mirroring Uniswap's standard approach (e.g., 30 minutes on Ethereum mainnet, 5 minutes on L2 networks). Additionally, consider allowing the protocol to dynamically adjust the deadline based on on-chain conditions, adapting to different requirements that may arise.

**Tools Used**

Manual Review
