# PoC 01 - Missing TSS Authorization

## Status

This is a simplified security model, not a confirmed vulnerability in the production Push Chain contracts.

## Related Production File

```text
contracts/evm-gateway/src/Vault.sol
```

## Target Function

```solidity
Vault.finalizeUniversalTx(...)
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

The external finalization function does not verify that the caller has the trusted TSS role.

Without this check, an attacker can directly submit attacker-controlled finalization data.

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

## What the PoC Proves

### Vulnerable Case

```text
Attacker
-> calls finalizeUniversalTx(...)
-> no TSS authorization is checked
-> attacker-controlled recipient and amount are accepted
```

The test proves the bad state:

```solidity
lastRecipient == ATTACKER
lastAmount == 1_000_000 ether
```

### Fixed Case

```text
Attacker
-> calls finalizeUniversalTx(...)
-> msg.sender is checked against TSS
-> call reverts with OnlyTSS()
```

`vm.expectRevert(...)` means that the test expects the protection to reject the attacker.

A passing fixed test means that the authorization check worked.

## Impact

An unauthorized caller could trigger fake destination-side execution using an attacker-controlled recipient or amount.

If finalization releases or transfers real assets, this could lead to unauthorized fund execution.

## Scope Note

The real Push Chain implementation protects `Vault.finalizeUniversalTx(...)` using TSS role authorization.

This PoC demonstrates why the invariant is important. It does not claim that the production Push Chain implementation is vulnerable.
