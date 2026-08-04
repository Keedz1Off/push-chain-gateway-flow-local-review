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

### _routeUniversalTx(...)

```text
Invariant:
1. _routeUniversalTx(...) must only be reached through the valid gateway flow.

Where I checked:

function _routeUniversalTx(
    UniversalTxRequest memory req,
    address caller,
    uint256 nativeValue,
    TX_TYPE txType,
    bool fromCEA
) internal {

Protection / Check:
internal <---

Status:
Protected

My reasoning:
Interval - can be called only inside this function or child contracts.
```
```text
Invariant:

2. Protocol fee must be collected when the call is not from CEA.

Where I checked:

   if (!fromCEA) {
        uint256 feeCollected;
        (nativeValue, feeCollected) = _collectInboundFee(nativeValue);
        totalProtocolFeesCollected += feeCollected;
    }

Protection / Check:

 uint256 feeCollected;
        (nativeValue, feeCollected) = _collectInboundFee(nativeValue);
        totalProtocolFeesCollected += feeCollected;

Status:

Protected.

My reasoning:
The fee was collected by _collectInboundFee(nativeValue), then the collected fee is saved to feeCollected.

Invariant:

3. txType must route the transaction to the correct internal function.


Where I checked:

 if (txType == TX_TYPE.GAS || txType == TX_TYPE.GAS_AND_PAYLOAD) {
        address gasRecipient = fromCEA ? req.recipient : address(0); //  result 1: req.recipient 
        _sendTxWithGas(
            txType, caller, gasRecipient, nativeValue, req.payload, req.revertRecipient, req.signatureData, fromCEA
        );
    } else if (txType == TX_TYPE.FUNDS || txType == TX_TYPE.FUNDS_AND_PAYLOAD) {
        _sendTxWithFunds(req, nativeValue, txType, fromCEA);
    } else {
        revert Errors.InvalidTxType();
    }
}

Protection / Check:

1. If txType is TX_TYPE.GAS or TX_TYPE.GAS_AND_PAYLOAD, the recipient is saved as req.recipient when caller is CEA and as address(0) if caller is not CEA; it is then forwarded to _sendTxWithGas()

2. If txType is TX_TYPE.FUNDS or TX_TYPE.FUNDS_AND_PAYLOAD, the recipient is saved as 


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
