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

What it does:

```text
TODO
```

Scope note:

```text
TODO: explain whether CEA implementation is inside or outside the current scope.
```

