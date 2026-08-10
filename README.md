# Push Chain Gateway Flow Local Review
<img width="1774" height="887" alt="62d0d14c-0453-4de2-9287-304bd824b8c6" src="https://github.com/user-attachments/assets/0edfe258-a5f5-402a-beb7-212bab799187" />
This repository is a local security review of the Push Chain Gateway flow.

The goal is to understand the architecture, follow the full cross-chain flow, and prepare invariant-based security notes.

## Contest Context

This review was prepared against the Push Chain EVM Gateway scope published for the Push Chain DualDefense audit contest.





The repository documents my independent contest-oriented review process:

```text
Trace the complete flow
-> identify trust boundaries
-> define invariants
-> verify protections in code
-> isolate suspicious zones
-> prepare PoC ideas
```

This is a portfolio review and not an official audit report from Push Chain or the contest organizers.

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

## Review Status

The main EVM gateway source and finalization flows were reviewed function by function. The completed notes currently contain:

```text
27 invariant checks
23 protected checks
4 suspicious checks grouped into 1 finding candidate
0 confirmed vulnerabilities
```

The remaining candidate concerns `UniversalTx` event data integrity. It requires a complete caller-to-event trace or a Foundry PoC before it can be classified as a confirmed vulnerability.

## Portfolio Scope

### Primary Reviews

```text
Push Chain Gateway contest-oriented review
Arbitrum bridge flow review
Optimism bridge flow review
LayerZero OFT flow review and exploit-lab practice
```

### Additional Architecture Studies

```text
Arbitrum L3 bridge architecture
Sky / DAI Optimism-based bridge fork
```

The primary reviews represent the main security practice in my portfolio. The L3 and Sky repositories are supporting architecture studies that show how bridge assumptions change across forks and layered systems.
