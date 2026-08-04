# Suspicious Zones

This file is for audit notes, not final findings.

Each section should explain why a part of the flow is security-sensitive and what I checked.

## Main Functions To Check

```text
sendUniversalTx(...)
_fetchTxType(...)
_routeUniversalTx(...)
_collectInboundFee(...)
_sendTxWithGas(...)
_sendTxWithFunds(...)
_handleDeposits(...)
_emitUniversalTx(...)
Vault.finalizeUniversalTx(...)
Vault._finalizeUniversalTx(...)
CEA.executeUniversalTx(...)
gateway.revertUniversalTx(...)
_validateRevertParams(...)
```

## 1. Event Data Integrity

Function:

```text
_emitUniversalTx(...)
```

Why it matters:

```text
TODO
```

What I checked:

```text
TODO
```

Risk:

```text
TODO
```

## 2. Funds Movement Before Event

Functions:

```text
_handleDeposits(...)
_sendTxWithFunds(...)
_sendTxWithGas(...)
```

Why it matters:

```text
TODO
```

What I checked:

```text
TODO
```

Risk:

```text
TODO
```

## 3. Finalize Authorization

Function:

```text
Vault.finalizeUniversalTx(...)
```

Why it matters:

```text
TODO
```

What I checked:

```text
TODO
```

Risk:

```text
TODO
```

## 4. Revert Recipient Integrity

Functions:

```text
_routeUniversalTx(...)
gateway.revertUniversalTx(...)
_validateRevertParams(...)
```

Why it matters:

```text
TODO
```

What I checked:

```text
TODO
```

Risk:

```text
TODO
```

## 5. Rate Limits And Caps

Functions:

```text
_consumeRateLimit(...)
_checkUSDCaps(...)
_checkBlockUSDCap(...)
```

Why it matters:

```text
TODO
```

What I checked:

```text
TODO
```

Risk:

```text
TODO
```

