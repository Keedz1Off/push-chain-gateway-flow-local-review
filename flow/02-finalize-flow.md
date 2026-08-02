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
TODO
```

Important checks:

```text
onlyRole(TSS_ROLE)
nonReentrant
whenNotPaused
```

Notes:

```text
TODO
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
TODO
```

Funds movement:

```text
ERC20: Vault -> CEA
native: Vault -> CEA through call value
```

Notes:

```text
TODO
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
TODO
```

Scope note:

```text
TODO: explain whether CEA implementation is inside or outside the current scope.
```

