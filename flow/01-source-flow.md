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

What it does:

```text
TODO
```

Notes:

```text
TODO
```

### _sendTxWithGas(...)

What it does:

```text
TODO
```

Notes:

```text
TODO
```

### _sendTxWithFunds(...)

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

What it does:

```text
TODO
```

Why it matters:

```text
TODO
```

