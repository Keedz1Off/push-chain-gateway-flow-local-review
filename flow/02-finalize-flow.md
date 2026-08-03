# 02 - Finalize Flow

## Flow

```text
off-chain TSS / relayer
-> Vault.finalizeUniversalTx(...)
-> CEAFactory.getCEAForPushAccount(...)
-> CEAFactory.deployCEA(...) if needed
-> Vault._finalizeUniversalTx(...)
-> CEA.executeUniversalTx(...)
```

## Functions To Review

### Vault.finalizeUniversalTx(...)

Function code:

```solidity
function finalizeUniversalTx(
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
```

What it does:

```text
Finalizes a universal transaction on the destination side.

Only TSS can call this function.
It gets the CEA address for pushAccount.
If CEA is not deployed yet, it deploys one.
Then it calls _finalizeUniversalTx(...).
```

Important checks:

```text
onlyRole(TSS_ROLE)
nonReentrant
whenNotPaused
```

Notes:

```text
This is the main destination-side entry point.
It does not directly execute the payload itself.
It prepares the correct CEA and passes execution to the internal finalize function.
The finalize event is emitted after the internal flow.
```

### Vault._finalizeUniversalTx(...)

Function code:

```solidity
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
        ICEA(cea).executeUniversalTx(subTxId, universalTxId, pushAccount, recipient, data);
    } else {
        ICEA(cea).executeUniversalTx{ value: amount }(subTxId, universalTxId, pushAccount, recipient, data);
    }
}
```

What it does:

```text
Moves funds to the CEA and calls executeUniversalTx(...).

For ERC20 tokens, Vault transfers tokens to CEA first.
For native token, Vault sends native value directly with the call.
```

Funds movement:

```text
ERC20: Vault -> CEA
native: Vault -> CEA through call value
```

Notes:

```text
_validateParams(...) checks basic input correctness.
If token is ERC20, Vault must have enough token balance.
If token is native, amount is passed as msg.value to CEA.
This function is the core finalization step.
```

### CEA.executeUniversalTx(...)

Function code:

```solidity
function executeUniversalTx(
    bytes32 subTxId,
    bytes32 universalTxId,
    address originCaller,
    address recipient,
    bytes calldata payload
) external payable;
```

What it does:

```text
Executes the final universal transaction from the CEA side.

Vault passes the transaction ids, original Push account, recipient, and payload to CEA.
```

Scope note:

```text
This file shows the interface call.
The important audit point in this flow is whether Vault passes the correct values into CEA.
```

