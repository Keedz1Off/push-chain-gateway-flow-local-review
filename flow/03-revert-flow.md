# 03 - Revert / Refund Flow

## Flow

```text
Vault.revertUniversalTx(...)
-> gateway.revertUniversalTx(...)
-> _validateRevertParams(...)
-> send funds to revertRecipient
```

## Functions To Review

### gateway.revertUniversalTx(...)

Function code:

```solidity
function revertUniversalTx(
    bytes32 subTxId,
    bytes32 universalTxId,
    address token,
    uint256 amount,
    RevertInstructions calldata revertInstruction
) external payable nonReentrant whenNotPaused onlyRole(VAULT_ROLE) {
    _validateRevertParams(subTxId, amount, token, revertInstruction.revertRecipient);

    if (token == address(0)) {
        (bool ok,) = payable(revertInstruction.revertRecipient).call{ value: amount }("");
        if (!ok) revert Errors.WithdrawFailed();
    } else {
        IERC20(token).safeTransfer(revertInstruction.revertRecipient, amount);
    }

    emit RevertUniversalTx(
        subTxId, universalTxId, revertInstruction.revertRecipient, token, amount, revertInstruction
    );
}
```

What it does:

```text
TODO
```

Important checks:

```text
onlyRole(VAULT_ROLE)
revertRecipient != address(0)
subTxId not executed
amount != 0
```

Notes:

```text
TODO
```

### _validateRevertParams(...)

Function code:

```solidity
function _validateRevertParams(bytes32 subTxId, uint256 amount, address token, address revertRecipient) private {
    if (isExecuted[subTxId]) revert Errors.PayloadExecuted();
    if (revertRecipient == address(0)) revert Errors.InvalidRecipient();
    if (amount == 0 || (token == address(0) && msg.value != amount)) revert Errors.InvalidAmount();

    isExecuted[subTxId] = true;
}
```

What it does:

```text
TODO
```

Replay protection:

```text
isExecuted[subTxId]
```

Notes:

```text
TODO
```

