### [H-03] Unbounded Loop in Consolidating Pending Stakes Disrupts Protocol

**Description**

A vulnerability within the smart contract allows a malicious user to inflate the [`LiquidationPool::pendingStakes`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L23) array excessively. The [`LiquidationPool::consolidatePendingStakes()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L119) function, crucial for [increasing](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L136), [decreasing](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L150) holders' positions, or [distributing assets](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L206) upon smart vault [liquidation](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L59), can cause [gas exhaustion](https://docs.soliditylang.org/en/latest/security-considerations.html#gas-limit-and-loops) and revert due to an unbounded loop.

```javascript
function consolidatePendingStakes() private {
    uint256 deadline = block.timestamp - 1 days;
@>  for (int256 i = 0; uint256(i) < pendingStakes.length; i++) { // @audit unbounded loop, reverts with high pendingStakes length
        PendingStake memory _stake = pendingStakes[uint256(i)];
        if (_stake.createdAt < deadline) {
            positions[_stake.holder].holder = _stake.holder;
            positions[_stake.holder].TST += _stake.TST;
            positions[_stake.holder].EUROs += _stake.EUROs;
            deletePendingStake(uint256(i));
            // Pause iterating on loop because there has been a deletion. "Next" item has the same index
            i--;
        }
    }
}
```

**Impact**

The vulnerability in [`LiquidationPool::consolidatePendingStakes()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L119) severely disrupts crucial functionalities of the contract. It affects the increase, decrease of stakes, and asset distribution during liquidation. Consequently, funds can be indefinitely locked within affected smart vaults. This issue not only hampers holders' transactions but also enables malicious borrowers to exploit the protocol by preventing liquidation.

**Proof of Concept**

1. Bob creates a smart vault, deposits 10 ETH as collateral, and mints 16000 EUROs.
2. Bob repeatedly calls [`LiquidationPool::increasePosition()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L134), continually escalating his position until the function reverts.
3. A day after Bob's deposits, market volatility causes ETH's price to drop, requiring liquidation of his vault.
4. A user initiates [`LiquidationPoolManager::runLiquidation()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L59), triggering [`LiquidationPool::distributeAssets()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L206). However, the [`LiquidationPool::consolidatePendingStakes()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L119) function reverts due to hitting the block gas limit, rendering Bob's vault inliquidatable.

**Recommended Mitigation**

Consider improving the consolidation logic for pending stakes or reworking the protocol's logic associated with them. Possible solutions include:

1. Introduce function parameters like `offset` and `length` to divide consolidation into smaller batches.
2. Implement an upper limit on the number of pending stakes.

**Tools Used**

Manual Review
