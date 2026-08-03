# Revert Flow Break Think

## gateway.revertUniversalTx(...)

```text
INVARIANT
1. Only Vault can trigger the revert flow.
2. Reverted funds must be sent to the intended revertRecipient.
3. RevertUniversalTx event data must match the real reverted funds.

CONSEQUENCES
1. This may lead to unauthorized refunds.
2. This may lead to refund going to the wrong address.
3. This may lead to off-chain systems reading wrong revert data.
```

## _validateRevertParams(...)

```text
INVARIANT
1. The same subTxId must not be reverted twice.
2. revertRecipient must not be address(0).
3. Native revert amount must match msg.value.

CONSEQUENCES
1. This may lead to double refund.
2. This may lead to funds being sent to an invalid recipient.
3. This may lead to incorrect native refund accounting.
```

