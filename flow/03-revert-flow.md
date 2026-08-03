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
Returns funds to the revert recipient when universal transaction must be reverted.

Only Vault can call this function.
It validates revert parameters, sends native or ERC20 funds, and emits RevertUniversalTx event.
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
Native token is sent with call value.
ERC20 token is sent with safeTransfer.
The revert recipient must be valid.
The function depends on _validateRevertParams(...) for replay protection.
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
Validates the revert request before funds are returned.

It checks that the subTxId was not already used.
It checks that revertRecipient is not address(0).
It checks that amount is not zero.
For native token, msg.value must equal amount.
Then it marks subTxId as executed.
```

Replay protection:

```text
isExecuted[subTxId]
```

Notes:

```text
isExecuted[subTxId] prevents the same revert from being processed twice.
The state is updated before funds are sent in the parent function.
This protects the refund path from replay.
```

