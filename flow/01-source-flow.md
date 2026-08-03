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
Receives user data, detects the transaction type, and routes the transaction to _routeUniversalTx(...).

This is the main user entry point on the source side.
```

Important parameters:

```text
req 
msg.value
```

Notes:

```text
CEA cannot call this function directly.
msg.value is saved as nativeValue.
The function does not move funds itself.
It only starts the source flow.
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
Detects what kind of transaction the user wants to send.

It checks if the request has payload, funds amount, native value, or ERC20 token.
Then it returns one of the TX_TYPE values.
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
GAS means only native value was sent.
GAS_AND_PAYLOAD means payload exists, but funds amount is zero.
FUNDS means funds are being sent without payload.
FUNDS_AND_PAYLOAD means funds and payload are sent together.

Invalid combinations revert with InvalidInput().
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
Routes the transaction into the correct internal path.

It checks the revert recipient, collects protocol fee for normal user flow, and then chooses either gas flow or funds flow.
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
revertRecipient must not be address(0).
If the call is not from CEA, the gateway collects INBOUND_FEE.
GAS and GAS_AND_PAYLOAD go to _sendTxWithGas(...).
FUNDS and FUNDS_AND_PAYLOAD go to _sendTxWithFunds(...).

fromCEA changes how recipient is handled.
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
Collects the inbound protocol fee from nativeValue.

If INBOUND_FEE is zero, nothing is collected.
If nativeValue is smaller than the fee, the transaction reverts.
The fee is sent to TSS_ADDRESS.
```

Notes:

```text
Returns two values:
adjustedNative = nativeValue after fee subtraction.
feeCollected = the protocol fee amount.

Example:
nativeValue = 100
fee = 5
return = (95, 5)
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
Handles the gas-only path.

If gas amount is greater than zero, it checks caps, deposits native value to TSS_ADDRESS, and emits UniversalTx event.
```

Notes:

```text
This path uses address(0) as token because it works with native value.
The event is important because off-chain TSS / relayers read it as the source message.
If _gasAmount is zero, it does not deposit funds but can still emit a payload event.
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
Handles the funds path.

It supports native funds, ERC20 funds, funds only, and funds with payload.
It may also split nativeValue into funds amount and gas amount.
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
For native funds, _req.amount must match the native value used as funds.
For ERC20 funds, nativeValue can be used as gas top-up.
Before deposit, the function consumes the rate limit.
Then it deposits funds through _handleDeposits(...).
Finally it emits UniversalTx event for off-chain processing.
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
Moves funds into the correct holding place.

Native token is sent to TSS_ADDRESS.
ERC20 token is transferred from the sender to VAULT.
```

Funds movement:

```text
native token -> TSS_ADDRESS
ERC20 token -> VAULT
```

Notes:

```text
address(0) means native token.
For ERC20, tokenToLimitThreshold[token] must be non-zero.
This means unsupported tokens are rejected.
This is the real source-side funds movement step.
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
Emits the UniversalTx event with all cross-chain transaction data.

The event contains sender, recipient, token, amount, payload, revert recipient, transaction type, signature data, and fromCEA flag.
```

Why it matters:

```text
Off-chain TSS / relayers use this event as the source-side message.

If event data does not match the real deposited funds or intended payload, the destination side may finalize the wrong action.
```

