# INV-6: The 1/64 kept back after a starved call may now be enough to finish

**Type:** Price

When a call runs out of gas, the caller keeps 1/64 of its gas (EIP-150). Code that was safe only because that 1/64
was too little to finish may now finish, and record a failure for good.

## Description

When a contract forwards all its gas to a call, the callee gets 63/64 and the caller keeps 1/64. If the call runs
out of gas, the caller continues with that 1/64.

Many contracts were safe *by accident*. A caller who wanted to make the call fail had to give it less gas than it
needed, and 1/64 of that was too little for the work after the call. So the whole transaction reverted.

For example, take a call that needs 300,000 gas:

- starving it leaves the caller under 4,700;
- that is too little even to update a storage slot, so the transaction reverts.

Glamsterdam raises what the callee needs, often because it now creates state at 97,920 a slot. The same call might
now need 1,000,000. Starving it leaves the caller about 15,600, which is enough to update a slot and emit an event.
The transaction succeeds, and the failure is recorded.

The range of gas in which this happens moves and widens. It also catches honest senders: a plain
`eth_estimateGas` only checks that the transaction does not revert, so it can land in exactly this range.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract TaskExecutor {
    struct Task { address target; bytes data; bool done; }
    mapping(uint256 => Task) public tasks;

    event Executed(uint256 indexed id, bool success);

    // Anyone can call this. Nothing checks the gas.
    function execute(uint256 id) external {
        Task storage t = tasks[id];
        require(!t.done, "done");
        t.done = true;                                // permanent

        (bool ok, ) = t.target.call(t.data);          // all gas forwarded; can be starved
        emit Executed(id, ok);                        // the kept-back 1/64 now pays for this
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract TaskExecutor {
    // minGas: what the call needs, with some headroom for the CALL itself (account access, memory).
    struct Task { address owner; address target; bytes data; uint256 minGas; bool done; }
    mapping(uint256 => Task) public tasks;

    event Executed(uint256 indexed id, bool success);

    // The task's owner knows what the call needs and can raise it when prices change.
    function setMinGas(uint256 id, uint256 minGas) external {
        require(msg.sender == tasks[id].owner, "not owner");
        tasks[id].minGas = minGas;
    }

    // Anyone can call this, with any gas.
    function execute(uint256 id) external {
        Task storage t = tasks[id];
        require(!t.done, "done");
        t.done = true;                                // before the call, so the target cannot re-enter

        // Read everything from storage first, so no gas is spent between the check and the call.
        address target = t.target;
        bytes memory data = t.data;
        uint256 minGas = t.minGas;

        // EIP-150: the callee gets at most 63/64 of the gas left. Revert unless it gets at least
        // minGas, so a failure below is the task's own and not a starved call.
        require(gasleft() >= minGas * 64 / 63, "not enough gas");
        (bool ok, ) = target.call(data);              // all gas forwarded: minGas is a floor, not a cap
        emit Executed(id, ok);
    }
}
```

The contract holds no gas number of its own. The gas a task needs belongs to the task, and its owner can raise it.
The executor can always send more: forwarding all gas means a floor that was set too low does not cap the call.
Where failing for good is not acceptable at all, revert on failure instead, as in INV-1.

## Common pitfalls

- **"The 1/64 is too small to matter."** It scales with what the callee needs, and that just went up.
- **Recording the outcome before or after an unchecked call:** nonces, "executed" flags, "processed" markers,
  burned receipts or NFTs.
- **Testing at one gas value.** The problem is a *range* of gas. Sweep from the lowest gas that does not revert up to
  well above the callee's need, and check the outcome at every step.
- **Trusting `eth_estimateGas`.** It finds a gas at which the transaction does not revert, not one at which the
  inner call succeeds. If failures are caught, make them revert during estimation (an estimation-only flag or
  sender), or check the result in simulation.

## Other things to consider

- **The fix is one of two things:**
  - make failures retryable instead of final (the simplest: revert on failure);
  - check before the call that the callee will get the gas it needs, allowing for the 63/64 rule. Take that amount
    from whoever knows the work and let them change it. A constant in the contract fails at the next repricing.
- **Watch functions that anyone can call:** relayers, keepers, bridge finalisation, multisig execution. The caller
  picks the gas there.
- **Related:** INV-1 (with a reservoir, the failure branch needs even less normal gas) and INV-7 (reserves sized for
  old prices).
