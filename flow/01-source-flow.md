# 01 - Source Flow

## Flow

```text
sendUniversalTx(...)
-> _fetchTxType(...)
-> _routeUniversalTx(...)
-> _collectInboundFee(...)
-> _sendTxWithGas(...) / _sendTxWithFunds(...)
-> _handleDeposits(...)
-> _emitUniversalTx(...)
```

## Functions To Review

### sendUniversalTx(...)

Function code:

```solidity
function sendUniversalTx(UniversalTxRequest calldata req) external payable nonReentrant whenNotPaused {
    if (_isCallerCEA()) revert Errors.InvalidInput();

    uint256 nativeValue = msg.value;

    TX_TYPE txType = _fetchTxType(req, nativeValue);

    _routeUniversalTx(req, _msgSender(), nativeValue, txType, false);
}
```

What it does:

```text
TODO: explain what this function does.
```

Important parameters:

```text
req
msg.value
```

Notes:

```text
TODO
```

### _fetchTxType(...)

Function code:

```solidity
function _fetchTxType(UniversalTxRequest memory req, uint256 nativeValue) private pure returns (TX_TYPE inferred) {
    bool hasPayload = req.payload.length > 0;
    bool hasFunds = req.amount > 0;
    bool fundsIsNative = (req.token == address(0));
    bool hasNativeValue = nativeValue > 0;

    if (!hasPayload && !hasFunds && hasNativeValue) {
        return TX_TYPE.GAS;
    }

    if (hasPayload && !hasFunds) {
        return TX_TYPE.GAS_AND_PAYLOAD;
    }

    if (!hasPayload && hasFunds) {
        if (fundsIsNative && hasNativeValue) {
            return TX_TYPE.FUNDS;
        }

        if (!fundsIsNative) {
            return TX_TYPE.FUNDS;
        }

        revert Errors.InvalidInput();
    }

    if (hasPayload && hasFunds) {
        if (!fundsIsNative) {
            return TX_TYPE.FUNDS_AND_PAYLOAD;
        }

        if (fundsIsNative && hasNativeValue) {
            return TX_TYPE.FUNDS_AND_PAYLOAD;
        }

        revert Errors.InvalidInput();
    }

    revert Errors.InvalidInput();
}
```

What it does:

```text
TODO
```

Checks:

```text
hasPayload
hasFunds
fundsIsNative
hasNativeValue
```

Notes:

```text
TODO
```

### _routeUniversalTx(...)

Function code:

```solidity
function _routeUniversalTx(
    UniversalTxRequest memory req,
    address caller,
    uint256 nativeValue,
    TX_TYPE txType,
    bool fromCEA
) internal {
    if (req.revertRecipient == address(0)) {
        revert Errors.InvalidRecipient();
    }

    if (!fromCEA) {
        uint256 feeCollected;
        (nativeValue, feeCollected) = _collectInboundFee(nativeValue);
        totalProtocolFeesCollected += feeCollected;
    }

    if (txType == TX_TYPE.GAS || txType == TX_TYPE.GAS_AND_PAYLOAD) {
        address gasRecipient = fromCEA ? req.recipient : address(0);
        _sendTxWithGas(
            txType, caller, gasRecipient, nativeValue, req.payload, req.revertRecipient, req.signatureData, fromCEA
        );
    } else if (txType == TX_TYPE.FUNDS || txType == TX_TYPE.FUNDS_AND_PAYLOAD) {
        _sendTxWithFunds(req, nativeValue, txType, fromCEA);
    } else {
        revert Errors.InvalidTxType();
    }
}
```

What it does:

```text
TODO
```

Important branches:

```text
GAS
GAS_AND_PAYLOAD
FUNDS
FUNDS_AND_PAYLOAD
```

Notes:

```text
TODO
```

### _collectInboundFee(...)

Function code:

```solidity
function _collectInboundFee(uint256 nativeValue) private returns (uint256 adjustedNative, uint256 feeCollected) {
    uint256 fee = INBOUND_FEE;
    if (fee == 0) return (nativeValue, 0);

    if (nativeValue < fee) revert Errors.InsufficientProtocolFee();

    (bool ok,) = payable(TSS_ADDRESS).call{ value: fee }("");
    if (!ok) revert Errors.DepositFailed();

    return (nativeValue - fee, fee);
}
```

What it does:

```text
TODO
```

Notes:

```text
TODO
```

### _sendTxWithGas(...)

Function code:

```solidity
function _sendTxWithGas(
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

    _emitUniversalTx(
        _caller, _recipient, address(0), _gasAmount, _payload, _revertRecipient, _txType, _signatureData, _fromCEA
    );
}
```

What it does:

```text
TODO
```

Notes:

```text
TODO
```

### _sendTxWithFunds(...)

Function code:

```solidity
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
```

What it does:

```text
TODO
```

Important branches:

```text
native funds
ERC20 funds
funds only
funds + payload
gas top-up
```

Notes:

```text
TODO
```

### _handleDeposits(...)

Function code:

```solidity
function _handleDeposits(address token, uint256 amount) internal {
    if (token == address(0)) {
        (bool ok,) = payable(TSS_ADDRESS).call{ value: amount }("");
        if (!ok) revert Errors.DepositFailed();
    } else {
        if (tokenToLimitThreshold[token] == 0) revert Errors.NotSupported();
        IERC20(token).safeTransferFrom(_msgSender(), VAULT, amount);
    }
}
```

What it does:

```text
TODO
```

Funds movement:

```text
native token -> TSS_ADDRESS
ERC20 token -> VAULT
```

Notes:

```text
TODO
```

### _emitUniversalTx(...)

Function code:

```solidity
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
```

What it does:

```text
TODO
```

Why it matters:

```text
TODO
```

