# Protected Invariants / No Issue Found

This file is for invariants that were checked and appear to be protected.

Use this format:

```text
Invariant: 

Where I checked:

Protection / Check:

Status:
Protected / Suspicious / Broken

My reasoning:
```

## Source Flow

### sendUniversalTx(...)

```text
Invariant:
_routeUniversalTx(...) must only be reached through the valid gateway flow.
Where I checked:
 ```solidity
function sendUniversalTx(UniversalTxRequest calldata req) external payable nonReentrant whenNotPaused {
    if (_isCallerCEA()) revert Errors.InvalidInput();
 ```

Protection / Check:

Status:

My reasoning:
```

### _fetchTxType(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _routeUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _collectInboundFee(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _sendTxWithGas(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _sendTxWithFunds(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _handleDeposits(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _emitUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

## Finalize Flow

### Vault.finalizeUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### Vault._finalizeUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### CEA.executeUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

## Revert / Refund Flow

### gateway.revertUniversalTx(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```

### _validateRevertParams(...)

```text
Invariant:

Where I checked:

Protection / Check:

Status:

My reasoning:
```
