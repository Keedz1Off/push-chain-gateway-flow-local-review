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

2. Protocol fee must be collected when the call is not from CEA.


Where I checked:

3. function _routeUniversalTx(
    UniversalTxRequest memory req,
    address caller,
    uint256 nativeValue,
    TX_TYPE txType,
    bool fromCEA
) internal {

2.  if (!fromCEA) {
        uint256 feeCollected;
        (nativeValue, feeCollected) = _collectInboundFee(nativeValue);
        totalProtocolFeesCollected += feeCollected;
    }


Protection / Check:
1. ()internal 


2. uint256 feeCollected;
        (nativeValue, feeCollected) = _collectInboundFee(nativeValue);
        totalProtocolFeesCollected += feeCollected;

3.  if (txType == TX_TYPE.GAS || txType == TX_TYPE.GAS_AND_PAYLOAD) {
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
Status:
1. Protected.

2. Protected.

3. Protected 

My reasoning:
1. Interval - can be called only inside this function or child contracts.

2. The fee was collected by _collectInboundFee(nativeValue), then the collected fee is saved to feeCollected.

3. If txType is TX_TYPE.GAS or TX_TYPE.GAS_AND_PAYLOAD, the recipient is saved as req.recipient when caller is CEA and as address(0) if caller is not CEA; it is then forwarded to _sendTxWithGas().
If txType is TX_TYPE.FUNDS or TX_TYPE.FUNDS_AND_PAYLOAD; it is then forwarded to _sendTxWithFunds().
Otherwise it reverts.


```


### _sendTxWithGas(...)

```text
Invariant:
1. Gas amount must be checked against the configured limits.

2. Native gas funds must be deposited to the correct TSS address.

Where I checked:

 1.function _sendTxWithGas(
    TX_TYPE _txType,
    address _caller,
    address _recipient,
    uint256 _gasAmount,
    bytes memory _payload,
    address _revertRecipient,
    bytes memory _signatureData,
    bool _fromCEA
) private {
    if (_gasAmount > 0) {
        _checkUSDCaps(_gasAmount);
        _checkBlockUSDCap(_gasAmount);
        _handleDeposits(address(0), _gasAmount);
    }
2. function _handleDeposits(address token, uint256 amount) internal {
    if (token == address(0)) {
        (bool ok,) = payable(TSS_ADDRESS).call{ value: amount }("");
        if (!ok) revert Errors.DepositFailed();
    } else {
        if (tokenToLimitThreshold[token] == 0) revert Errors.NotSupported();
        IERC20(token).safeTransferFrom(_msgSender(), VAULT, amount);
    }
}


3.

   // this is a part of  _sendTxWithGas()



emitUniversalTx(
        _caller, _recipient, address(0), _gasAmount, _payload, _revertRecipient, _txType, _signatureData, _fromCEA
    );
}

          


Protection / Check:

 1. if (_gasAmount > 0) {
        _checkUSDCaps(_gasAmount);
        _checkBlockUSDCap(_gasAmount);
        _handleDeposits(address(0), _gasAmount);
    }
2.  if (token == address(0)) {
        (bool ok,) = payable(TSS_ADDRESS).call{ value: amount }("");
        if (!ok) revert Errors.DepositFailed();


3. 

Status:
1.Protected.

2.Protected.

3.Protected.

My reasoning:

1. If this funcion receives _gasAmount, the function first checks the gas limits using checkUSDCaps(), the second check verifies the gas amount limit per the block.After all call. Finally _handleDeposits() is called.


2. When _handleDeposits() is called , the contract sends native funds to TSS_ADDRESS using payable.


3. 



```


### _sendTxWithFunds(...)

```text


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
