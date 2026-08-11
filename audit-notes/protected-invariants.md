# Protected Invariants / No Issue Found

This file is for invariants that were checked and appear to be protected.

## Review Summary

```text
Completed function reviews: 9
Checked invariants: 27
Protected checks: 23
Suspicious checks: 4
Distinct suspicious finding candidates: 1
Confirmed vulnerabilities: 0
```

The suspicious checks point to one shared candidate: event data integrity in `_emitUniversalTx(...)` and its caller `_sendTxWithGas(...)`.

Current status: **Suspicious / not yet confirmed**.

This becomes a confirmed finding only if a caller can make the emitted `UniversalTx` data differ from the funds actually deposited or from the user's intended request. The absence of validation inside `_emitUniversalTx(...)` alone is not sufficient evidence because this helper may rely on validation performed earlier in the flow.

## Contents

- Source Flow
- Finalize Flow
- Revert / Refund Flow
- Review Conclusion

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

3. txType must route the transaction to the correct internal function.


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

3. The emitted UniversalTx data must match the gas flow data.


Where I checked:

1. function _sendTxWithGas(
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

Full function:

          function _emitUniversalTx(
    address sender,
    address recipient,
    address token,
    uint256 amount,
    bytes memory payload,
    address revertRecipient,
    TX_TYPE txType,
    bytes memory signatureData,
    bool fromCEA
) private {
    emit UniversalTx({
        sender: sender,
        recipient: recipient,
        token: token,
        amount: amount,
        payload: payload,
        revertRecipient: revertRecipient,
        txType: txType,
        signatureData: signatureData,
        fromCEA: fromCEA
    });
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


3.   emit UniversalTx({
        sender: sender,
        recipient: recipient,
        token: token,
        amount: amount,
        payload: payload,
        revertRecipient: revertRecipient,
        txType: txType,
        signatureData: signatureData,
        fromCEA: fromCEA
    });
}

Status:
1. Protected.

2. Protected.

3. Suspicious

My reasoning:

1. If this funcion receives _gasAmount, the function first checks the gas limits using checkUSDCaps(), the second check verifies the gas amount limit per the block.After all call. Finally _handleDeposits() is called.


2. When _handleDeposits() is called , the contract sends native funds to TSS_ADDRESS using payable.Otherwise it reverts.


3. The UniversalTx() is risky because _emitUniversalTx() don't verifies data before emitting,this may lead to native funds going to the wrong address by changing emit data.




```


### _sendTxWithFunds(...)

```text
Invariant:
1. Native funds amount must match the native value used in the transaction.

2. ERC20 funds must be deposited before UniversalTx is emitted.

3. FUNDS_AND_PAYLOAD must preserve both funds data and payload data.

Where I checked:

function _sendTxWithFunds(UniversalTxRequest memory _req, uint256 nativeValue, TX_TYPE txType, bool fromCEA)
    private
{
    if (txType == TX_TYPE.FUNDS) {
        address tokenForFunds;

        if (_req.token == address(0)) {
            if (_req.amount != nativeValue) revert Errors.InvalidAmount();
            tokenForFunds = address(0);
        } else {
            if (nativeValue > 0) {
                address gasRecipient = fromCEA ? _req.recipient : address(0);
                _sendTxWithGas(
                    TX_TYPE.GAS,
                    _msgSender(),
                    gasRecipient,
                    nativeValue,
                    bytes(""),
                    _req.revertRecipient,
                    _req.signatureData,
                    fromCEA
                );
            }
            tokenForFunds = _req.token;
        }

        _consumeRateLimit(tokenForFunds, _req.amount);
        _handleDeposits(tokenForFunds, _req.amount);

        _emitUniversalTx(
            _msgSender(),
            _req.recipient,
            tokenForFunds,
            _req.amount,
            _req.payload,
            _req.revertRecipient,
            txType,
            _req.signatureData,
            fromCEA
        );
    }

    if (txType == TX_TYPE.FUNDS_AND_PAYLOAD) {
        address tokenForFundsAndPayload;
        address gasRecipient = fromCEA ? _req.recipient : address(0);

        if (nativeValue == 0) {
            if (_req.token == address(0)) revert Errors.InvalidAmount();
            tokenForFundsAndPayload = _req.token;
        } else if (_req.token == address(0)) {
            if (nativeValue < _req.amount) revert Errors.InvalidAmount();

            uint256 gasAmount = nativeValue - _req.amount;

            if (gasAmount > 0) {
                _sendTxWithGas(
                    TX_TYPE.GAS,
                    _msgSender(),
                    gasRecipient,
                    gasAmount,
                    bytes(""),
                    _req.revertRecipient,
                    _req.signatureData,
                    fromCEA
                );
            }
            tokenForFundsAndPayload = address(0);
        } else if (_req.token != address(0)) {
            uint256 gasAmount = nativeValue;
            _sendTxWithGas(
                TX_TYPE.GAS,
                _msgSender(),
                gasRecipient,
                gasAmount,
                bytes(""),
                _req.revertRecipient,
                _req.signatureData,
                fromCEA
            );

            tokenForFundsAndPayload = _req.token;
        }

        _consumeRateLimit(tokenForFundsAndPayload, _req.amount);
        _handleDeposits(tokenForFundsAndPayload, _req.amount);

        address fundsAndPayloadRecipient = fromCEA ? _req.recipient : address(0);
        _emitUniversalTx(
            _msgSender(),
            fundsAndPayloadRecipient,
            tokenForFundsAndPayload,
            _req.amount,
            _req.payload,
            _req.revertRecipient,
            txType,
            _req.signatureData,
            fromCEA
        );
    }
}

Protection / Check:

1. if (_req.token == address(0)) {
       if (_req.amount != nativeValue) revert Errors.InvalidAmount();
   }

2. _handleDeposits(tokenForFunds, _req.amount);
   _emitUniversalTx(...);

3. _emitUniversalTx(
       _msgSender(),
       fundsAndPayloadRecipient,
       tokenForFundsAndPayload,
       _req.amount,
       _req.payload,
       _req.revertRecipient,
       txType,
       _req.signatureData,
       fromCEA
   );

Status:
1. Protected.

2. Protected.

3. Protected.

My reasoning:

Assistant Review Note:
1. In the native FUNDS path, _req.amount must equal nativeValue. In the native FUNDS_AND_PAYLOAD path, nativeValue must cover _req.amount and the remaining value is separated as gasAmount.

2. _handleDeposits(...) is called before _emitUniversalTx(...). If the deposit fails, the transaction reverts and the event is not emitted.

3. The selected token, _req.amount, and _req.payload are forwarded into the same UniversalTx event. This keeps the funds and payload together in the FUNDS_AND_PAYLOAD path.
```


### _handleDeposits(...)

```text
Invariant:
1. Native funds must be sent to TSS_ADDRESS.

2. ERC20 funds must be transferred to VAULT.

3. Unsupported ERC20 tokens must be rejected.

Where I checked:
1,2,3. function _handleDeposits(address token, uint256 amount) internal {
    if (token == address(0)) {
        (bool ok,) = payable(TSS_ADDRESS).call{ value: amount }("");
        if (!ok) revert Errors.DepositFailed();
    } else {
        if (tokenToLimitThreshold[token] == 0) revert Errors.NotSupported();
        IERC20(token).safeTransferFrom(_msgSender(), VAULT, amount);
    }
}


Protection / Check:

1.  if (token == address(0)) {
        (bool ok,) = payable(TSS_ADDRESS).call{ value: amount }("");
        if (!ok) revert Errors.DepositFailed()

2.  IERC20(token).safeTransferFrom(_msgSender(), VAULT, amount);



3.  if (tokenToLimitThreshold[token] == 0) revert Errors.NotSupported()


Status:

1. Protected

2. Protected

3. Protected 


My reasoning:
1. When _handleDeposits() is called, the contract sends native funds to TSS_ADDRESS using payable.Otherwise it reverts ( Errors.DepositFailed() ).

2. When  _handleDeposits() is called, the contract  sends token (ERC20) funds to VAULT using and safeTransfer(). Otherwise it reverts. Errors.NotSupported()  . 

3. if token is not found in mapping, it reverts ( Errors.NotSupported() ).



```

### _emitUniversalTx(...)

```text
Invariant:

1. Event amount must match the real deposited amount.

2. Event token must match the real deposited token.

3. Event recipient and payload must match the intended user request.


Where I checked:

1,2,3. function _emitUniversalTx(
    address sender,
    address recipient,
    address token,
    uint256 amount,
    bytes memory payload,
    address revertRecipient,
    TX_TYPE txType,
    bytes memory signatureData,
    bool fromCEA
) private {
    emit UniversalTx({
        sender: sender,
        recipient: recipient,
        token: token,
        amount: amount,
        payload: payload,
        revertRecipient: revertRecipient,
        txType: txType,
        signatureData: signatureData,
        fromCEA: fromCEA
    });
}


Protection / Check:
 1. emit UniversalTx({
        amount: amount,
    });

2.   emit UniversalTx({
       token: token,
    });

3. emit UniversalTx({
       payload: payload,

       recipient: recipient,
         
    });




Status:

1,2,3. Suspicious.




My reasoning:

1,2,3. The UniversalTx() is risky because _emitUniversalTx() don't verifies data before emitting,this may lead to native funds going to the wrong address by changing emit data.

```

## Finalize Flow

### Vault.finalizeUniversalTx(...)

```text
Invariant:
1. Only TSS can finalize a universal transaction.

2. The correct CEA must be used for the pushAccount.

3. Finalize data must not be changed before _finalizeUniversalTx(...).

Where I checked:

1. function finalizeUniversalTx(
    bytes32 subTxId,
    bytes32 universalTxId,
    address pushAccount,
    address recipient,
    address token,
    uint256 amount,
    bytes calldata data
) external payable nonReentrant whenNotPaused onlyRole(TSS_ROLE) {
    (address cea, bool isDeployed) = CEAFactory.getCEAForPushAccount(pushAccount);
    if (!isDeployed) {
        cea = CEAFactory.deployCEA(pushAccount);
    }

    _finalizeUniversalTx(subTxId, universalTxId, pushAccount, recipient, token, amount, data, cea);

    emit UniversalTxFinalized(subTxId, universalTxId, pushAccount, recipient, token, amount, data);
}

Protection / Check:

1.    onlyRole(TSS_ROLE)

2.     (address cea, bool isDeployed) = CEAFactory.getCEAForPushAccount(pushAccount);
    if (!isDeployed) {
        cea = CEAFactory.deployCEA(pushAccount);

3.     _finalizeUniversalTx(subTxId, universalTxId, pushAccount, recipient, token, amount, data, cea) 



Status:

1. Protected.

2. Protected.

3. Protected.


My reasoning:

1. Only TSS can call this function.


2. The Vault asks CEAFactory for the CEA address that belongs to this pushAccount, if the CEA address is not found; then CEAFactory.deployCEA deploys a new CEA for this pushAccount.

3.  The finalizeUniversalTx() parameters  match the _finalizeUniversalTx()  parameters.
```

### Vault._finalizeUniversalTx(...)

```text
Invariant:
1. ERC20 amount must be transferred from Vault to the correct CEA before execution.

2. Native amount must be sent to CEA as call value before execution.

3. CEA execution parameters must match the finalized transaction data.


Where I checked:
function _finalizeUniversalTx(
    bytes32 subTxId,
    bytes32 universalTxId,
    address pushAccount,
    address recipient,
    address token,
    uint256 amount,
    bytes calldata data,
    address cea
) private {
    _validateParams(pushAccount, token, amount);

    if (token != address(0)) {
        if (amount > 0) {
            if (IERC20(token).balanceOf(address(this)) < amount) {
                revert Errors.InvalidAmount();
            }
            IERC20(token).safeTransfer(cea, amount);
        }
        ICEA(cea).executeUniversalTx(subTxId, universalTxId, pushAccount, recipient, data); // ERC20 token
    } else {
        ICEA(cea).executeUniversalTx{ value: amount }(subTxId, universalTxId, pushAccount, recipient, data); // native token
    }
}
Protection / Check:

1. if (token != address(0)) {
        if (amount > 0) {
            if (IERC20(token).balanceOf(address(this)) < amount) {
                revert Errors.InvalidAmount();
            }
            IERC20(token).safeTransfer(cea, amount);
        }
        ICEA(cea).executeUniversalTx(subTxId, universalTxId, pushAccount, recipient, data); // ERC20 token



2.   } else {
        ICEA(cea).executeUniversalTx{ value: amount }(subTxId, universalTxId, pushAccount, recipient, data); // native token




3.  _validateParams(pushAccount, token, amount);


Status:
1.  Protected.

2.  Protected.

3. Protected.

My reasoning:
1. Vault checks that it has enought ERC20 tokens and uses safeTransfer() to send exact amount to the CEA before .executeUniversalTx() is called. Otherwise it reverts 

2. Vault sends native tokens to CEA.

3. _validateParams() verifies the  parameters ( pushAccount, token, amount ) before sending them to the CEA.Otherwise it reverts 

```

## Revert / Refund Flow

### gateway.revertUniversalTx(...)

```text
Invariant:

1. Only Vault can trigger the revert flow.

2. Reverted funds must be sent to the intended revertRecipient.

3. RevertUniversalTx event data must match the real reverted funds

Where I checked:

1.

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

Protection / Check:

1. onlyRole(VAULT_ROLE

2.  _validateRevertParams(subTxId, amount, token, revertInstruction.revertRecipient);

     (bool ok,) = payable(revertInstruction.revertRecipient).call{ value: amount }("");
        if (!ok) revert Errors.WithdrawFailed();

3.  _validateRevertParams(subTxId, amount, token, revertInstruction.revertRecipient);

Status:

1. Protected.

2. Protected.

3. Protected. 

My reasoning:

1. Only Vault can call this function.

2. Vault verifies revertRecipient using _validateRevertParams and sends the funds to it.

3. Vault verifies data for RevertUniversalTx() using  _validateRevertParams().

```

### _validateRevertParams(...)

```text
Invariant:

1. The same subTxId must not be reverted twice.

2. revertRecipient must not be address(0).

3. Native revert amount must match msg.value.

Where I checked:

function _validateRevertParams(bytes32 subTxId, uint256 amount, address token, address revertRecipient) private {
    if (isExecuted[subTxId]) revert Errors.PayloadExecuted();
    if (revertRecipient == address(0)) revert Errors.InvalidRecipient();
    if (amount == 0 || (token == address(0) && msg.value != amount)) revert Errors.InvalidAmount();

    isExecuted[subTxId] = true;
}

Protection / Check:

1.  if (isExecuted[subTxId]) revert Errors.PayloadExecuted();

2.  if (revertRecipient == address(0)) revert Errors.InvalidRecipient();

3.  if (amount == 0 || (token == address(0) && msg.value != amount)) revert Errors.InvalidAmount();


Status:

1. Protected.

2. Protected. 

3. Protected.

My reasoning:

1. It checks that the subTxId was not already used. Otherwise it reverts.

2. It checks that amount is not zero.For native token, msg.value must equal amount.Otherwise it reverts.

3. For native token, msg.value must equal amount.Then it marks subTxId as executed.


```

## Review Conclusion

### Result

```text
One suspicious finding candidate was identified.
No vulnerability is confirmed yet.
```

### Candidate: UniversalTx Event Data Integrity

```text
Status: Suspicious / requires a PoC or a complete caller-to-event trace.

Affected area:
_sendTxWithGas(...)
_emitUniversalTx(...)

Question to prove:
Can user-controlled or incorrectly routed data make the emitted token, amount, recipient, or payload differ from the funds actually deposited and the intended request?

Required next step:
Build a Foundry test that compares the real deposit state with the emitted UniversalTx fields. If they cannot diverge, mark the candidate as Protected. If they can diverge and the off-chain TSS trusts the event, write it as a confirmed finding.
```
