# Finalize Flow Break Think

## Vault.finalizeUniversalTx(...)

```text
INVARIANT
1. Only TSS can finalize a universal transaction.
2. The correct CEA must be used for the pushAccount.
3. Finalize data must be passed to the internal finalize flow without unintended modification.

CONSEQUENCES
1. This may lead to fake finalization.
2. This may lead to execution through the wrong CEA.
3. This may lead to wrong recipient, token, amount, or payload being finalized.
```

## Vault._finalizeUniversalTx(...)

```text
INVARIANT
1. ERC20 amount must be transferred from Vault to the correct CEA before execution.
2. Native amount must be sent to CEA as call value before execution.
3. CEA execution parameters must match the finalized transaction data.

CONSEQUENCES
1. This may lead to execution without correct ERC20 funds.
2. This may lead to execution without correct native funds.
3. This may lead to wrong destination execution.
```

## CEA.executeUniversalTx(...)

```text
INVARIANT
1. CEA must execute the transaction for the correct origin caller.
2. CEA must send the payload to the intended recipient.
3. CEA execution must use the finalized payload data.

CONSEQUENCES
1. This may lead to execution for the wrong source account.
2. This may lead to funds or call execution going to the wrong recipient.
3. This may lead to wrong payload execution on the destination side.
```

