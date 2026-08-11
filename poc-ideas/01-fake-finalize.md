# 01 - Fake Finalize

## Related Production File

```text
contracts/evm-gateway/src/Vault.sol
```

## Target Function

```solidity
Vault.finalizeUniversalTx(...)
```

## Status

```text
Simplified PoC model.
Not a confirmed vulnerability in the production Push Chain contracts.
```

## Broken Invariant

```text
Only TSS can finalize a universal transaction.
```

## Consequence

```text
This may lead to fake finalization.
```

## Root Cause

The vulnerable version does not verify that `msg.sender` has the trusted TSS role.

An attacker can therefore call `finalizeUniversalTx(...)` directly and provide an attacker-controlled recipient and amount.

## Vulnerable Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableVault {
    address public lastRecipient;
    uint256 public lastAmount;

    function finalizeUniversalTx(
        address recipient,
        uint256 amount
    ) external {
        // BUG: missing TSS authorization

        lastRecipient = recipient;
        lastAmount = amount;
    }
}
```

## Fixed Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract FixedVault {
    error OnlyTSS();

    address public immutable tss;

    address public lastRecipient;
    uint256 public lastAmount;

    constructor(address _tss) {
        tss = _tss;
    }

    function finalizeUniversalTx(
        address recipient,
        uint256 amount
    ) external {
        if (msg.sender != tss) {
            revert OnlyTSS();
        }

        lastRecipient = recipient;
        lastAmount = amount;
    }
}
```

## Foundry PoC Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract MissingTSSAuthorizationPoC is Test {
    address internal constant TSS = address(0x7001);
    address internal constant ATTACKER = address(0xBAD);

    VulnerableVault internal vulnerableVault;
    FixedVault internal fixedVault;

    function setUp() public {
        vulnerableVault = new VulnerableVault();
        fixedVault = new FixedVault(TSS);
    }

    function test_Vulnerable_AttackerCanFinalize() public {
        vm.prank(ATTACKER);

        vulnerableVault.finalizeUniversalTx(
            ATTACKER,
            1_000_000 ether
        );

        assertEq(
            vulnerableVault.lastRecipient(),
            ATTACKER
        );

        assertEq(
            vulnerableVault.lastAmount(),
            1_000_000 ether
        );
    }

    function test_Fixed_AttackerCannotFinalize() public {
        vm.prank(ATTACKER);

        vm.expectRevert(
            FixedVault.OnlyTSS.selector
        );

        fixedVault.finalizeUniversalTx(
            ATTACKER,
            1_000_000 ether
        );
    }

    function test_Fixed_TSSCanFinalize() public {
        vm.prank(TSS);

        fixedVault.finalizeUniversalTx(
            ATTACKER,
            100 ether
        );

        assertEq(
            fixedVault.lastRecipient(),
            ATTACKER
        );

        assertEq(
            fixedVault.lastAmount(),
            100 ether
        );
    }
}
```

## Attack Flow

```text
Attacker
-> directly calls finalizeUniversalTx(...)
-> no TSS authorization is checked
-> attacker-controlled recipient and amount are accepted
-> fake finalization succeeds
```

## What the Vulnerable Test Proves

The attacker directly calls `finalizeUniversalTx(...)`.

The vulnerable contract accepts the following fake data:

```text
recipient = ATTACKER
amount = 1,000,000 tokens
```

The test proves the bad state using:

```solidity
assertEq(
    vulnerableVault.lastRecipient(),
    ATTACKER
);

assertEq(
    vulnerableVault.lastAmount(),
    1_000_000 ether
);
```

A passing vulnerable test means that the invariant violation was reproduced.

## What the Fixed Test Proves

The fixed contract checks:

```solidity
if (msg.sender != tss) {
    revert OnlyTSS();
}
```

The test expects the attacker call to revert:

```solidity
vm.expectRevert(
    FixedVault.OnlyTSS.selector
);
```

A passing fixed test means that the TSS authorization successfully blocked the attacker.

## Impact

Without TSS authorization, an attacker may be able to submit fake finalization data.

Depending on the real destination execution logic, this may lead to:

```text
- fake finalization;
- execution for the wrong recipient;
- processing of an incorrect amount;
- execution of an attacker-controlled payload;
- unauthorized release or transfer of funds.
```

## Production Protection

The real Push Chain `Vault.finalizeUniversalTx(...)` is protected by TSS role authorization.

Conceptually, the protection is:

```solidity
onlyRole(TSS_ROLE)
```

Therefore, this PoC explains the importance of the authorization invariant. It does not claim that the current production contract is vulnerable.
