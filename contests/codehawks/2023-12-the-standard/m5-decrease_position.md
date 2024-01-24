### [M-05] Unavailable Staked Amount Retrieval for Holders with Pending Positions

**Description**

Holders can withdraw a portion or the entire amount of their staked assets using the [`LiquidationPool::decreasePosition`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L149) method. However, access to this method is restricted to holders with positions older than 1 day. This limitation arises due to the [`LiquidationPool::consolidatePendingStakes()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPool.sol#L119) function, which checks the age of pending stakes to ensure fairer depositing procedures.

```javascript
function decreasePosition(uint256 _tstVal, uint256 _eurosVal) external {
    consolidatePendingStakes(); // @audit only position > 1 day are consolidated
    ILiquidationPoolManager(manager).distributeFees();
@>  require(_tstVal <= positions[msg.sender].TST && _eurosVal <= positions[msg.sender].EUROs, "invalid-decr-amount"); // @audit will fail for newly created position
    if (_tstVal > 0) {
        IERC20(TST).safeTransfer(msg.sender, _tstVal);
@>      positions[msg.sender].TST -= _tstVal;
    }
    if (_eurosVal > 0) {
        IERC20(EUROs).safeTransfer(msg.sender, _eurosVal);
@>      positions[msg.sender].EUROs -= _eurosVal;
    }
    if (empty(positions[msg.sender])) deletePosition(positions[msg.sender]);
}
```

**Impact**

This behavior not only significantly impacts a holder's funds, especially in times of market volatility or when a user has recently deposited funds, but it also compromises the user experience. Mistakenly sent funds urgently required by the user will remain locked for an entire day, potentially leading to frustration and dissatisfaction with the platform's functionality. Such limitations can erode user trust and confidence in the system's reliability and usability, undermining the platform's overall reputation and hindering its adoption and success.

**Proof of Concept**

A demonstration illustrating a holder's inability to withdraw their funds can be found [here](https://gist.github.com/DanailYordanov/4745d725c06bd414686575df20c6861f) using Foundry.

**Recommended Mitigation**

To address this issue, consider adding a new function explicitly designed to retrieve all pending stakes associated with a user. This new functionality would enable the system to comprehensively check and retrieve all pending stakes belonging to a user, allowing them to withdraw all pending stakes in a single action.

```javascript
function deleteAllPendingStakes(address holder) private {
    for (int256 i = 0; uint256(i) < pendingStakes.length; i++) {
        if (pendingStakes[uint256(i)].holder == holder) {
            deletePendingStake(uint256(i));
            i--;
        }
    }
}

function withdrawPendingStakes() external {
    (uint256 _pendingTST, uint256 _pendingEUROs) = holderPendingStakes(msg.sender);
    require(_pendingTST > 0 || _pendingEUROs > 0, "no-pending-stakes");
    if (_pendingTST > 0) IERC20(TST).safeTransfer(msg.sender, _pendingTST);
    if (_pendingEUROs > 0) IERC20(EUROs).safeTransfer(msg.sender, _pendingEUROs);
    deleteAllPendingStakes(msg.sender);
}
```

**Tools Used**

Manual review
