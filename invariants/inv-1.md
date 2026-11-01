# INV-1: A call that runs out of gas no longer takes the whole transaction down

**Type:** Accounting

If a transaction carries a storage-gas reservoir, the code after a failed call can still write storage on very little
normal gas. A caller can starve a call on purpose and have the contract record a failure it cannot undo.

## Description

Many contracts make a call and then record the outcome:

```solidity
(bool ok, ) = target.call(data);
if (!ok) failed[id] = true;
```

Before Glamsterdam, a caller who starved the call left the contract only the 1/64 of gas that EIP-150 keeps back.
Writing a new slot costs 22,100, so that 1/64 was often too little for the failure branch. The whole transaction
reverted and could be retried. Many contracts are safe today only because of this, often by accident.

EIP-8037 splits the cost of creating state (new slots, new accounts, new contracts) into a separate *state gas*.
When the transaction's gas limit is above 2^24 (16,777,216), the part above that goes into a **reservoir**. State
gas is paid from the reservoir first, and only then from normal gas.

The reservoir is not limited by `call{gas: X}` or by the 63/64 rule. It passes to every called contract in full.
An attacker can therefore:

1. send a transaction with a gas limit above 2^24, which creates a reservoir;
2. call your contract through a contract of their own with a small, exact amount of normal gas;
3. let the inner call run out of gas.

Your failure branch then pays only the normal-gas part of its writes (about 12,100 per new slot). The 97,920 of state
gas per slot comes from the reservoir. The failure branch completes, and the failure is recorded for good.

The attacker needs a balance that covers the high gas limit up front. Unused gas is refunded, so the attack itself
is cheap.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract MessageProcessor {
    struct Message { address target; bytes data; }

    mapping(uint256 => Message) public messages;
    mapping(uint256 => bool) public processed;
    mapping(uint256 => bool) public failed;

    // Anyone can call this, with any gas.
    function process(uint256 id) external {
        require(!processed[id], "already processed");
        processed[id] = true;                         // permanent: no retry

        Message memory m = messages[id];
        (bool ok, ) = m.target.call(m.data);          // can be starved on purpose

        if (!ok) {
            failed[id] = true;                        // new slot: state gas comes from the reservoir
        }
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract MessageProcessor {
    struct Message { address target; bytes data; uint256 gasLimit; }

    // Normal gas needed after the call. Size it at Amsterdam prices.
    uint256 public constant POST_CALL_GAS = 50_000;

    mapping(uint256 => Message) public messages;
    mapping(uint256 => bool) public processed;

    event Failed(uint256 indexed id);

    function process(uint256 id) external {
        require(!processed[id], "already processed");
        Message memory m = messages[id];

        // EIP-150: the callee gets at most 63/64 of what is left. Revert, rather than
        // record a failure, if the caller did not supply enough gas.
        require(gasleft() * 63 / 64 >= m.gasLimit + POST_CALL_GAS, "not enough gas");

        (bool ok, ) = m.target.call{gas: m.gasLimit}(m.data);

        if (ok) {
            processed[id] = true;
        } else {
            emit Failed(id);                          // nothing permanent: the message can be retried
        }
    }
}
```

## Common pitfalls

- **Treating `call{gas: X}` as a limit on everything the callee can do.** It limits normal gas only. A callee can
  create state with the whole reservoir.
- **"The attacker cannot afford more than 2^24 gas."** They only need the balance to cover the gas limit up front.
  Unused gas is refunded.
- **Testing only below 2^24.** The behaviour above it differs, so test both sides.
- **Pre-call gas checks that count normal gas only.** `gasleft()` does not include the reservoir. A check that
  passes says nothing about how much state the callee can create.
- **"The failure branch only emits an event."** Check what else it does. Writing a new slot, bumping a counter that
  was zero, or creating an account all count.

## Other things to consider

- **Prefer outcomes that can be retried.** Recording a failure is fine if someone can try again. It is the
  permanent "done, failed" state that turns a starved call into a loss.
- **Callers can pick the gas** in relayers, keepers, bridge executors, batchers and any function that anyone can
  call. Look there first.
- **The rule may change.** A proposed spec change (EIPs PR #12581) makes state creation take part of its cost from
  normal gas even when a reservoir exists. That narrows this attack but does not remove it.
- **Related:** INV-3 (gas handed back by a reverting call can pay for the failure branch) and INV-6 (the 1/64 kept
  back may now be enough).
