# 02 - Event Amount Mismatch

## Related Production File

```text
contracts/evm-gateway/src/UniversalGateway.sol
```

## Target Functions

```solidity
_sendTxWithFunds(...)
_handleDeposits(...)
_emitUniversalTx(...)
```

## Status

```text
Simplified PoC model.
Not a confirmed vulnerability in the production Push Chain contracts.
```

## Broken Invariant

```text
Event amount must match the real deposited amount.
```

## Consequence

```text
This may lead to finalizing a different amount on the destination side.
```

## Root Cause

The vulnerable version accepts the deposited amount and the emitted event amount as two independent values.

It does not verify that the amount written into the cross-chain event equals the amount actually deposited into the gateway.

If an off-chain TSS or relayer trusts the event, it may process an amount that is not backed by the real source-chain deposit.

## Vulnerable Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableEventGateway {
    uint256 public depositedAmount;
    uint256 public emittedAmount;

    event UniversalTx(
        address indexed sender,
        uint256 amount
    );

    function sendUniversalTx(
        uint256 amountForEvent
    ) external payable {
        depositedAmount += msg.value;

        // BUG: amountForEvent is not checked against msg.value
        emittedAmount = amountForEvent;

        emit UniversalTx(
            msg.sender,
            amountForEvent
        );
    }
}
```

## Fixed Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract FixedEventGateway {
    error AmountMismatch();

    uint256 public depositedAmount;
    uint256 public emittedAmount;

    event UniversalTx(
        address indexed sender,
        uint256 amount
    );

    function sendUniversalTx(
        uint256 amountForEvent
    ) external payable {
        if (amountForEvent != msg.value) {
            revert AmountMismatch();
        }

        depositedAmount += msg.value;
        emittedAmount = amountForEvent;

        emit UniversalTx(
            msg.sender,
            amountForEvent
        );
    }
}
```

## Foundry PoC Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract EventAmountMismatchPoC is Test {
    VulnerableEventGateway internal vulnerableGateway;
    FixedEventGateway internal fixedGateway;

    function setUp() public {
        vulnerableGateway = new VulnerableEventGateway();
        fixedGateway = new FixedEventGateway();
    }

    function test_Vulnerable_EventAmountCanDifferFromDeposit() public {
        uint256 realDeposit = 1 ether;
        uint256 fakeEventAmount = 100 ether;

        vulnerableGateway.sendUniversalTx{
            value: realDeposit
        }(
            fakeEventAmount
        );

        assertEq(
            vulnerableGateway.depositedAmount(),
            realDeposit
        );

        assertEq(
            vulnerableGateway.emittedAmount(),
            fakeEventAmount
        );

        assertTrue(
            vulnerableGateway.emittedAmount()
                != vulnerableGateway.depositedAmount()
        );
    }

    function test_Fixed_MismatchedAmountReverts() public {
        uint256 realDeposit = 1 ether;
        uint256 fakeEventAmount = 100 ether;

        vm.expectRevert(
            FixedEventGateway.AmountMismatch.selector
        );

        fixedGateway.sendUniversalTx{
            value: realDeposit
        }(
            fakeEventAmount
        );
    }

    function test_Fixed_CorrectAmountSucceeds() public {
        uint256 realDeposit = 1 ether;

        fixedGateway.sendUniversalTx{
            value: realDeposit
        }(
            realDeposit
        );

        assertEq(
            fixedGateway.depositedAmount(),
            realDeposit
        );

        assertEq(
            fixedGateway.emittedAmount(),
            realDeposit
        );
    }
}
```

## Attack Flow

```text
User deposits 1 native token
-> gateway records a real deposit of 1 token
-> event is created with an unchecked amount of 100 tokens
-> off-chain TSS or relayer reads 100 tokens
-> destination flow may attempt to finalize 100 tokens
```

## What the Vulnerable Test Proves

The source-side contract receives:

```text
msg.value = 1 ether
```

However, the event records:

```text
amount = 100 ether
```

The test confirms both values:

```solidity
assertEq(
    vulnerableGateway.depositedAmount(),
    1 ether
);

assertEq(
    vulnerableGateway.emittedAmount(),
    100 ether
);
```

It then proves that the invariant is broken:

```solidity
assertTrue(
    vulnerableGateway.emittedAmount()
        != vulnerableGateway.depositedAmount()
);
```

A passing vulnerable test means that the mismatch was successfully reproduced.

## What the Fixed Test Proves

The fixed version binds the event amount to the real deposit:

```solidity
if (amountForEvent != msg.value) {
    revert AmountMismatch();
}
```

When the event amount differs from the deposited value, the transaction reverts.

When both amounts are equal, the transaction succeeds.

## Impact

If the destination flow trusts the incorrect event amount, the mismatch may lead to:

```text
- finalizing a different amount;
- processing unbacked funds;
- incorrect cross-chain accounting;
- loss of funds;
- insolvency of the bridge or Vault.
```

The final severity depends on whether the TSS or destination contract independently verifies the deposited amount.

## Production Analysis

In the reviewed Push Chain flow, `_emitUniversalTx(...)` does not perform independent validation.

However, this does not automatically mean the production code is vulnerable.

For the native `FUNDS` flow, the caller checks:

```solidity
if (_req.amount != nativeValue) {
    revert Errors.InvalidAmount();
}
```

The real deposit then happens before the event:

```solidity
_handleDeposits(
    tokenForFunds,
    _req.amount
);

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
```

Both operations use the same `_req.amount`.

If `_handleDeposits(...)` fails, the entire transaction reverts and the event is not emitted.

## Current Conclusion

```text
Suspicious zone: event data integrity
Confirmed vulnerability: no
Current protection: caller-side amount validation and atomic execution
```

This PoC demonstrates what would happen if the deposited amount and emitted amount were no longer bound together.

It should be treated as an invariant-break exercise, not as a confirmed Push Chain finding.
