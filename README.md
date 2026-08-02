# Push Chain Gateway Flow Local Review

This repository is a local security review of the Push Chain Gateway flow.

The goal is to understand the architecture, follow the full cross-chain flow, and prepare invariant-based security notes.

## Scope

```text
UniversalGateway.sol
Vault.sol
ICEA.sol
ICEAFactory.sol
Types.sol
TypesUG.sol
```

## Main Flow

```text
User
-> UniversalGateway.sendUniversalTx(...)
-> _fetchTxType(...)
-> _routeUniversalTx(...)
-> _sendTxWithGas(...) / _sendTxWithFunds(...)
-> _handleDeposits(...)
-> _emitUniversalTx(...)
-> off-chain TSS / relayer
-> Vault.finalizeUniversalTx(...)
-> Vault._finalizeUniversalTx(...)
-> CEA.executeUniversalTx(...)
```

## Folder Structure

```text
flow/
  Function-by-function flow notes.

break-think/
  Invariant templates. I will fill these manually.

audit-notes/
  Suspicious zones and security review notes.

poc-ideas/
  PoC ideas and test plans.

glossary/
  Important words and concepts.
```

## Review Method

```text
Understand the flow
-> Track funds and payload
-> Define invariants
-> Check suspicious zones
-> Write PoC ideas
```
