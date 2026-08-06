# Source Flow Break Think

## _routeUniversalTx(...)

```text
INVARIANT
1. _routeUniversalTx(...) must only be reached through the valid gateway flow.
2. Protocol fee must be collected when the call is not from CEA.
3. txType must route the transaction to the correct internal function.

CONSEQUENCES
1. This may lead to bypassing the normal gateway flow.
2. This may lead to protocol fee bypass.
3. This may lead to funds or payload being processed through the wrong flow.
```

## _sendTxWithGas(...)

```text
INVARIANT
1. Gas amount must be checked against the configured limits.
2. Native gas funds must be deposited to the correct TSS address.
3. The emitted UniversalTx data must match the gas flow data.

CONSEQUENCES
1. This may lead to bypassing gas limits.
2. This may lead to native funds going to the wrong address.
3. This may lead to off-chain relayers reading wrong gas transaction data.
```

## _sendTxWithFunds(...)

```text
INVARIANT
1. Native funds amount must match the native value used in the transaction.
2. ERC20 funds must be deposited before UniversalTx is emitted.
3. FUNDS_AND_PAYLOAD must preserve both funds data and payload data.

CONSEQUENCES
1. This may lead to wrong amount being processed.
2. This may lead to message creation without real deposited funds.
3. This may lead to destination execution with wrong funds or wrong payload.
```

## _handleDeposits(...)

```text
INVARIANT
1. Native funds must be sent to TSS_ADDRESS.
2. ERC20 funds must be transferred to VAULT.
3. Unsupported ERC20 tokens must be rejected.

CONSEQUENCES
1. This may lead to native funds going to the wrong address.
2. This may lead to ERC20 funds being stuck or not escrowed correctly.
3. This may lead to unsupported token deposits.
```

## _emitUniversalTx(...)

```text
INVARIANT
1. Event amount must match the real deposited amount.
2. Event token must match the real deposited token.
3. Event recipient and payload must match the intended user request.

CONSEQUENCES
1. This may lead to finalizing a different amount on the destination side.
2. This may lead to finalizing the wrong token.
3. This may lead to funds or payload being executed for the wrong recipient.
```

