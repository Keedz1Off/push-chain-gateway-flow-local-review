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

