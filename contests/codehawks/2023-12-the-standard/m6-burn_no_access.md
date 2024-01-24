### [M-06] No Access Control in Burning EUROs from Smart Vault Can Lead to Falsified Off-Chain Data

**Description**

The [`SmartVaultV3::burn()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L169) method lacks access control measures, allowing anyone to burn minted EUROs instead of restricting this functionality to the owner. A potential exploit involves a malicious user creating a Smart Vault, depositing collateral, and minting EUROs. Subsequently, the attacker could call the [`SmartVaultV3::burn()`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L169) method on someone else's Smart Vault, which also has minted EUROs. This action would alter the [`SmartVaultV3::minted`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L27) variable, providing false data compared to the actual holdings.

```javascript
function burn(uint256 _amount) external ifMinted(_amount) { // @audit lacks access control, missing "onlyOwner" modifier
    uint256 fee = _amount * ISmartVaultManagerV3(manager).burnFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    minted = minted - _amount; // @audit vulnerability: potential for unauthorized manipulation of 'minted'
    EUROs.burn(msg.sender, _amount);
    IERC20(address(EUROs)).safeTransferFrom(msg.sender, ISmartVaultManagerV3(manager).protocol(), fee);
    emit EUROsBurned(_amount, fee);
}
```

**Impact**

While this vulnerability doesn't result in fund loss for the victim or any direct advantage for the attacker, it can significantly disrupt off-chain representations of the current minted EUROs. This discrepancy could lead to a poor user experience due to misleading or falsified data representations.

**Proof of Concept**

The Foundry test demonstrating the disruption of the [`SmartVaultV3::minted`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L27) storage variable is expected to be found [here](https://gist.github.com/DanailYordanov/8fb7b09100c12000a43506fcf540f54f).

**Recommended Mitigation**

To address this issue, it's essential to restrict the burning of EUROs to only the owner by adding the [`SmartVaultV3::onlyOwner`](https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultV3.sol#L48) modifier:

```diff
- function burn(uint256 _amount) external ifMinted(_amount) {
+ function burn(uint256 _amount) external onlyOwner ifMinted(_amount) {
    uint256 fee = _amount * ISmartVaultManagerV3(manager).burnFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
    minted = minted - _amount;
    EUROs.burn(msg.sender, _amount);
    IERC20(address(EUROs)).safeTransferFrom(msg.sender, ISmartVaultManagerV3(manager).protocol(), fee);
    emit EUROsBurned(_amount, fee);
}
```

**Tools Used**

Manual Review
