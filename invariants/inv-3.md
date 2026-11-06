# INV-3: Gas used by a call that reverts is not necessarily gone

**Type:** Accounting

When a call reverts, the state gas it spent creating storage is handed back to the caller, so the caller has more gas
left than it expects. Code that guesses "the callee ran out of gas" from the gas left, or from empty revert data, is
misled. So is code that does not check whether a CREATE succeeded.

## Description

Under EIP-8037, a reverted call creates no state. Any state gas it charged is credited back: what was paid from the reservoir returns to the reservoir, and on REVERT
what was paid from normal gas returns to the caller's normal gas (a call that runs out of gas returns no normal gas). This includes state created by contracts it called in turn. A failed CREATE or CREATE2 works the same way: the
183,600 charged for the new account comes back when the init code fails.

That breaks two common habits.

**Guessing why a call failed.** Contracts often treat a failure as an out-of-gas, and revert so the operation can be
retried, when:

- the revert data is empty (`reason.length == 0`, `returndatasize() == 0`); or
- little gas is left (`gasleft() <= gasBefore / 63`).

Now an intermediate contract can write some new slots, call something that runs out of gas, and revert. Its state gas
comes back to you, so `gasleft()` looks healthy. If that intermediate contract wraps the error, for example
OpenZeppelin `Address` or `SafeERC20` turning an empty revert into a custom error, the revert data is not empty
either. Your code takes the "real failure" branch and commits a permanent fallback.

**Unchecked contract creation.** Code that runs `pop(create2(...))` and then calls the expected address assumes the
deployment worked. If the deployment ran out of gas, the address has no code. A low-level call to an address with no
code succeeds and does nothing.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

interface IRegistry { function register(address validator) external; }

contract Depositor {
    IRegistry public immutable registry;
    mapping(address => uint256) public deposits;

    constructor(IRegistry r) { registry = r; }

    // Anyone can call this, with any gas.
    function flush(address validator) external {
        try registry.register(validator) {
            // registered
        } catch (bytes memory reason) {
            // Assumption: empty revert data means out of gas, so revert and let someone retry.
            if (reason.length == 0) revert("out of gas, retry");

            // Assumption: anything else is a real rejection, so refund for good.
            uint256 amount = deposits[validator];
            deposits[validator] = 0;
            payable(validator).transfer(amount);
        }
    }
}
```

`register` writes new slots, then calls a helper that runs out of gas and wraps the out-of-gas in a custom error. The
state gas comes back to `flush`, and `reason` is not empty. The validator is refunded and dropped instead of retried.

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

interface IRegistry {
    function canRegister(address validator) external view returns (bool);
    function register(address validator) external;
}

contract Depositor {
    IRegistry public immutable registry;
    mapping(address => uint256) public deposits;

    constructor(IRegistry r) { registry = r; }

    // Anyone can call this, with any gas.
    function flush(address validator) external {
        if (registry.canRegister(validator)) {
            // No try/catch: any failure reverts, so the deposit stays and anyone can retry with more gas.
            registry.register(validator);
        } else {
            // The registry said no. The permanent branch rests on that answer, not on a failed call.
            uint256 amount = deposits[validator];
            deposits[validator] = 0;
            (bool sent, ) = payable(validator).call{value: amount}("");
            require(sent, "refund failed");
        }
    }
}
```

The decision to refund no longer depends on why a call failed, so nothing here assumes how much gas `register`
needs. A starved `canRegister` reverts the whole transaction; it does not return `false`. If the registry has no such
check, revert on every failure and let the depositor take the deposit back in a separate transaction.

For contract creation, check the result:

```solidity
address deployed;
assembly { deployed := create2(0, add(code, 0x20), mload(code), salt) }
require(deployed != address(0) && deployed.code.length > 0, "deploy failed");
```

## Common pitfalls

- **"Empty revert data means out of gas."** Wrappers add data, and a reverted intermediate contract hands gas back.
- **Ratio checks like `gasleft() <= gasBefore / 63`.** The gas handed back can push `gasleft()` above the threshold.
- **Vyper `raw_call(..., revert_on_failure=False)` and low-level calls** whose result is used to pick a permanent
  branch.
- **`pop(create(...))` / `pop(create2(...))`**, or ignoring a zero address, followed by a call to the predicted
  address. The call to an empty address succeeds.
- **Token designs that deploy helper contracts on first use** (per-holder storage contracts) without checking the
  deployment.

## Other things to consider

- **After the call, the gas left tells you nothing reliable** about why it failed. Do not take a permanent branch on
  a failure. Revert so the operation can be retried, base the branch on an explicit answer (as above), or make sure
  before the call that the callee will receive the gas it needs (as in INV-6).
- **The rule is under discussion.** Proposed spec changes would send the handed-back gas to the reservoir instead
  (EIPs PR #12581), or burn it. Either would remove much of this. Do not rely on today's behaviour, in either
  direction.
- **Related:** INV-1 (failure branches paid by the reservoir) and INV-6 (the 1/64 kept back).
