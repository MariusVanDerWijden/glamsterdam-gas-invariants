# INV-2: Clearing a slot set earlier in the transaction gives gas back at once

**Type:** Accounting

A storage slot that was zero when the transaction started, is set, and is cleared again (0 → x → 0) now refills gas
immediately, so `gasleft()` goes up. That refill can pay for a failure branch that would otherwise run out of gas.

## Description

Before Glamsterdam, clearing a slot earned a refund paid out only at the end of the transaction, and capped at a
fifth of the gas used. During execution, `gasleft()` never went up.

Under EIP-8037, creating a slot costs 97,920 of state gas. If that slot is cleared again in the same transaction, no
state was created, so the state gas is given back straight away:

- first to the current frame's normal gas, up to what that frame itself paid from normal gas;
- the rest to the reservoir (see INV-1).

This means `gasleft()` can rise mid-function. Code that sets a flag, makes a call and clears the flag gets about
98,000 gas back after the call. If the call was starved on purpose, that gas pays for writing the failure.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Relay {
    mapping(address => uint256) public inFlight;      // 0 when idle
    mapping(bytes32 => bool) public failedMessages;

    function relay(address target, bytes calldata message) external {
        inFlight[msg.sender] = 1;                     // new slot: 97,920 state gas paid from normal gas

        (bool ok, ) = target.call(message);           // starved on purpose

        delete inFlight[msg.sender];                  // 0 -> 1 -> 0: about 98,000 gas comes back into gasleft()

        if (!ok) {
            failedMessages[keccak256(message)] = true; // paid by the refill; a failure is recorded for good
        }
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Relay {
    // Transient storage costs no state gas and refills nothing.
    bool transient private inFlight;

    uint256 public constant MESSAGE_GAS = 200_000;
    uint256 public constant POST_CALL_GAS = 50_000;

    event Failed(bytes32 indexed messageHash);

    function relay(address target, bytes calldata message) external {
        require(!inFlight, "reentrant");
        inFlight = true;

        // Revert instead of recording a failure when the caller supplied too little gas.
        require(gasleft() * 63 / 64 >= MESSAGE_GAS + POST_CALL_GAS, "not enough gas");
        (bool ok, ) = target.call{gas: MESSAGE_GAS}(message);

        inFlight = false;

        if (!ok) {
            emit Failed(keccak256(message));          // retryable: nothing permanent
        }
    }
}
```

## Common pitfalls

- **Locks whose idle value is zero.** A reentrancy lock that goes 0 → 1 → 0 refills on every use. OpenZeppelin's
  `ReentrancyGuard` rests at 1 and goes 1 → 2 → 1, which creates no state and refills nothing.
- **Temporary approvals.** Code that approves a router, calls it, then resets the allowance to zero refills if the
  allowance was zero when the transaction started.
- **In-flight flags and context slots** around an external call, such as "current sender" or "current message id".
  These are common in bridges and messengers.
- **Loops like `while (gasleft() > X)`** whose body clears a slot can run more times than expected.
- **Measuring gas with two `gasleft()` readings.** The difference can be negative. See INV-5.

## Other things to consider

- **Transient storage (EIP-1153) is the cleanest fix** for locks and per-call context. Solidity supports
  `transient` state variables of value types from 0.8.28.
- **Only slots that were zero when the transaction started refill.** A slot that was 5 and goes 5 → 7 → 5 does not.
- **The rule is under discussion.** A proposed spec change (EIPs PR #12581) sends these refills to the reservoir
  instead of normal gas. Then `gasleft()` would no longer rise, though the gas could still pay for state created
  later. Do not rely on either behaviour.
