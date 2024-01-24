### [M-02] ERC20 Tokens with Over 18 Decimals Causing Locked Funds

**Description**

While the protocol is compatible with tokens containing 18 decimals or fewer, it encounters issues with tokens having more than 18 decimals, such as [`YAMv2`](https://etherscan.io/token/0xaba8cac6866b83ae4eec97dd07ed254282f6ad8a) with 24 decimals. The susceptible part of the code resides within [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L205), specifically on [line 220](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L220). Here, the `costInEuros` variable is calculated with 18 decimal places by scaling the `_portion` variable with `10 ** (18 - asset.token.dec)`. This calculation is prone to underflow when dealing with tokens having more than 18 decimals.
```javascript
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
    // ** code ** 
    uint256 _portion = asset.amount * _positionStake / stakeTotal;
@>  uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd) * _hundredPC / _collateralRate; // @audit underflow if asset.token.dec is more than 18
    // ** code **
}
```

**Impact**

The impact of this vulnerability extends to the liquidation process within the protocol. As the [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L205) function is invoked during liquidation in the [`LiquidationPoolManager::runLiquidation()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L59), encountering assets with over 18 decimals causes the liquidation to revert. Consequently, smart vaults holding such assets won't undergo liquidation. This issue leads to an immediate loss of funds, as the liquidation process, crucial for securing funds, fails to execute properly.

**Proof of Concept**

A demonstration using Foundry showcasing the reversion of [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L205) when employing the `YAM` mock token can be accessed [here](https://gist.github.com/DanailYordanov/901b699b8ce35acc605c0d72a2f47d3a).

**Recommended Mitigation**

Consider augmenting the code to upscale and downscale the `_portion` variable, ensuring it maintains a consistent 18 decimal format without encountering underflow issues.

```diff
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
    // ** code ** 
    uint256 _portion = asset.amount * _positionStake / stakeTotal;
+   int8 decimalsDifference = 18 - int8(asset.token.dec);

+   if (decimalsDifference > 0) {
+       _portion = _portion * (10 ** uint256(int256(decimalsDifference)));
+   } else if (decimalsDifference < 0) {
+       _portion = _portion / (10 ** uint256(int256(-decimalsDifference)));
+   } 
-  uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd) * _hundredPC / _collateralRate;
+  uint256 costInEuros = _portion * uint256(assetPriceUsd) / uint256(priceEurUsd) * _hundredPC / _collateralRate;
    // ** code **
}
```

**Tools Used**

Manual Review