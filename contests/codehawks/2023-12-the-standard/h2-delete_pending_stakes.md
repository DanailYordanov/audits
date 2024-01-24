### [H-02] Unbounded Loop in Deleting Pending Stakes Leading to DOS Attack

**Description**

An exploit exists within the smart contract that allows a malicious user to inflate the [`LiquidationPool::pendingStakes`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L23) array excessively. Triggering the [`LiquidationPool::deletePendingStake()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L105) function with a large array size can cause [gas exhaustion](https://docs.soliditylang.org/en/latest/security-considerations.html#gas-limit-and-loops), leading to a revert due to exceeding the block gas limit. This issue occurs particularly when attempting to delete earlier indexes within the array.
```javascript
function deletePendingStake(uint256 _i) private {
@>  for (uint256 i = _i; i < pendingStakes.length - 1; i++) { //@audit unbounded loop
        pendingStakes[i] = pendingStakes[i+1];
    }
    pendingStakes.pop();
}
```
**Impact**

The vulnerability carries substantial impact due to the centrality of the pending stakes deletion process within the contract's operations. The [`LiquidationPool::consolidatePendingStakes()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L119) method, critical for the contract, relies on deleting pending stakes. The vulnerability disrupts essential functionalities including stake [increase](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L136), [decrease](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L150), and [asset distribution](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L206), which is pivotal in the [liquidation process](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L59) of smart vaults. This critical vulnerability renders several crucial functions dysfunctional, resulting in the potential indefinite lockup of funds within affected smart vaults.

**Proof of Concept**

1. Bob creates a smart vault, deposits 1 ETH as collateral, and mints 1600 EUROs.
2. Alice, a malicious user, repeatedly calls [`LiquidationPool::increasePosition()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L134), continuously escalating her position until the function reverts.
3. A day after Alice's deposits, market volatility causes ETH's price to drop and Bob's vault requires liquidation.
4. A user initiates [`LiquidationPoolManager::runLiquidation()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L59), triggering [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L206). This process relies on consolidating pending stakes which tries to delete the first pending stake, but the [`LiquidationPool::deletePendingStake()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L105) function reverts due to the block gas limit, making Bob's vault inliquidatable.

**Recommended Mitigation**

Consider improving the logic used to delete pending stakes. An effective solution involves moving the last element into the desired deletion position:
```javascript
function deletePendingStakes(uint256 _i) public {
    pendingStakes[_i] = pendingStakes[pendingStakes.length - 1];
    pendingStakes.pop();
}
```
**Tools Used**

Manual Review