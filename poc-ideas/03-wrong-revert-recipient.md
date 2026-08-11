# 03 - Wrong Revert Recipient

## Related Production Files

```text
contracts/evm-gateway/src/UniversalGateway.sol
contracts/evm-gateway/src/Vault.sol
```

## Target Functions

```solidity
gateway.revertUniversalTx(...)
_validateRevertParams(...)
```

## Status

```text
Simplified PoC model.
Not a confirmed vulnerability in the production Push Chain contracts.
```

## Broken Invariant

```text
Reverted funds must be sent to the intended revertRecipient.
```

## Consequence

```text
This may lead to refund going to the wrong address.
```

## Root Cause

The vulnerable version accepts `revertRecipient` from the caller without verifying that it matches the recipient associated with the original `subTxId`.

An attacker can provide their own address and redirect the refund.

## Vulnerable Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableRevertGateway {
    mapping(bytes32 => bool) public reverted;

    function revertUniversalTx(
        bytes32 subTxId,
        address payable revertRecipient
    ) external payable {
        // BUG: revertRecipient is not checked against subTxId.

        reverted[subTxId] = true;

        (bool success, ) = revertRecipient.call{
            value: msg.value
        }("");

        require(success, "REFUND_FAILED");
    }
}
```

## Fixed Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract FixedRevertGateway {
    error InvalidRevertRecipient();
    error AlreadyReverted();

    mapping(bytes32 => bool) public reverted;

    mapping(bytes32 => address)
        public intendedRevertRecipient;

    function registerRevertRecipient(
        bytes32 subTxId,
        address recipient
    ) external {
        intendedRevertRecipient[subTxId] = recipient;
    }

    function revertUniversalTx(
        bytes32 subTxId,
        address payable revertRecipient
    ) external payable {
        if (reverted[subTxId]) {
            revert AlreadyReverted();
        }

        if (
            revertRecipient == address(0) ||
            revertRecipient !=
                intendedRevertRecipient[subTxId]
        ) {
            revert InvalidRevertRecipient();
        }

        reverted[subTxId] = true;

        (bool success, ) = revertRecipient.call{
            value: msg.value
        }("");

        require(success, "REFUND_FAILED");
    }
}
```

## Foundry PoC Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract WrongRevertRecipientPoC is Test {
    address internal constant ATTACKER =
        address(0xBAD);

    address internal constant VICTIM =
        address(0xBEEF);

    VulnerableRevertGateway
        internal vulnerableGateway;

    FixedRevertGateway
        internal fixedGateway;

    function setUp() public {
        vulnerableGateway =
            new VulnerableRevertGateway();

        fixedGateway =
            new FixedRevertGateway();

        vm.deal(address(this), 10 ether);
    }

    function test_Vulnerable_RefundGoesToAttacker()
        public
    {
        bytes32 subTxId =
            keccak256("push-sub-tx-1");

        uint256 attackerBalanceBefore =
            ATTACKER.balance;

        uint256 victimBalanceBefore =
            VICTIM.balance;

        vulnerableGateway.revertUniversalTx{
            value: 1 ether
        }(
            subTxId,
            payable(ATTACKER)
        );

        assertEq(
            ATTACKER.balance,
            attackerBalanceBefore + 1 ether
        );

        assertEq(
            VICTIM.balance,
            victimBalanceBefore
        );
    }

    function test_Fixed_WrongRecipientIsRejected()
        public
    {
        bytes32 subTxId =
            keccak256("push-sub-tx-2");

        fixedGateway.registerRevertRecipient(
            subTxId,
            VICTIM
        );

        vm.expectRevert(
            FixedRevertGateway
                .InvalidRevertRecipient
                .selector
        );

        fixedGateway.revertUniversalTx{
            value: 1 ether
        }(
            subTxId,
            payable(ATTACKER)
        );
    }

    function test_Fixed_IntendedRecipientGetsRefund()
        public
    {
        bytes32 subTxId =
            keccak256("push-sub-tx-3");

        fixedGateway.registerRevertRecipient(
            subTxId,
            VICTIM
        );

        uint256 victimBalanceBefore =
            VICTIM.balance;

        fixedGateway.revertUniversalTx{
            value: 1 ether
        }(
            subTxId,
            payable(VICTIM)
        );

        assertEq(
            VICTIM.balance,
            victimBalanceBefore + 1 ether
        );

        assertEq(
            fixedGateway.reverted(subTxId),
            true
        );
    }

    function test_Fixed_SameTransactionCannotBeRevertedTwice()
        public
    {
        bytes32 subTxId =
            keccak256("push-sub-tx-4");

        fixedGateway.registerRevertRecipient(
            subTxId,
            VICTIM
        );

        fixedGateway.revertUniversalTx{
            value: 1 ether
        }(
            subTxId,
            payable(VICTIM)
        );

        vm.expectRevert(
            FixedRevertGateway
                .AlreadyReverted
                .selector
        );

        fixedGateway.revertUniversalTx{
            value: 1 ether
        }(
            subTxId,
            payable(VICTIM)
        );
    }
}
```

## Attack Flow

```text
Original transaction
-> intended revertRecipient is VICTIM
-> transaction fails
-> attacker supplies ATTACKER as revertRecipient
-> vulnerable gateway does not verify the recipient
-> refund is transferred to ATTACKER
```

## What the Vulnerable Test Proves

The test sends `1 ether` into the revert flow and supplies the attacker as the refund recipient.

The bad state is proven using:

```solidity
assertEq(
    ATTACKER.balance,
    attackerBalanceBefore + 1 ether
);
```

The intended recipient does not receive the refund:

```solidity
assertEq(
    VICTIM.balance,
    victimBalanceBefore
);
```

A passing vulnerable test means that the refund-redirection scenario was reproduced.

## What the Fixed Tests Prove

The fixed contract binds the intended recipient to `subTxId`:

```solidity
intendedRevertRecipient[subTxId]
```

Before transferring funds, it checks:

```solidity
revertRecipient ==
    intendedRevertRecipient[subTxId]
```

A wrong recipient causes:

```solidity
revert InvalidRevertRecipient();
```

The fixed version also blocks a second refund for the same transaction:

```solidity
if (reverted[subTxId]) {
    revert AlreadyReverted();
}
```

## Impact

If the refund recipient is not bound to the original transaction, reverted funds may be redirected to an attacker-controlled address.

Possible consequences:

```text
- refund sent to the wrong address;
- permanent loss of user funds;
- unauthorized refund;
- double refund if replay protection is also missing.
```

## Production Protection

The production implementation should verify:

```text
- only an authorized contract can trigger the revert flow;
- revertRecipient is valid;
- revertRecipient belongs to the original transaction;
- subTxId has not already been reverted;
- native amount matches msg.value.
```

This PoC demonstrates the importance of these invariants. It does not claim that the current production Push Chain contracts are vulnerable.
