# INV-5: `gasleft()` no longer measures what the work cost

**Type:** Accounting

`gasleft()` can now go up within a function, and state gas paid from the reservoir never shows up in it at all. Code
that charges or refunds by subtracting two `gasleft()` readings can underflow, over-pay or under-charge.

## Description

A common pattern for metering work:

```solidity
uint256 start = gasleft();
doWork();
uint256 used = start - gasleft();
```

Before Glamsterdam this was a reasonable estimate of what `doWork()` cost. Under EIP-8037 it breaks in two ways.

**1. `gasleft()` can rise.** Two things can push it up during `doWork()`:

- a slot set and cleared in the same transaction refills gas at once (see INV-2);
- when a called contract returns successfully, part of its unused state gas can move back into the caller's normal
  gas.

The later reading can then be *larger* than the earlier one:

- checked arithmetic reverts, which blocks the operation;
- `unchecked` arithmetic wraps around to a huge number, which can over-pay.

**2. Reservoir-paid state gas is invisible.** When the transaction's gas limit is above 2^24, state gas is paid from
the reservoir, not from normal gas. `gasleft()` only shows normal gas, so new slots, accounts and code paid from the
reservoir cost nothing as far as the meter can see. A contract that bills users this way under-charges them, and the
relayer or operator pays the difference.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract GasRefundingExecutor {
    uint256 public constant OVERHEAD = 21_000;

    // A relayer calls this and is repaid for the gas used, from the user's deposit.
    function executeFor(address user, address target, bytes calldata data) external {
        uint256 start = gasleft();

        (bool ok, ) = target.call(data);
        require(ok, "call failed");

        uint256 used;
        unchecked {
            used = start - gasleft() + OVERHEAD;      // wraps if gasleft() rose
        }
        _chargeUser(user, used * tx.gasprice);        // and misses reservoir-paid state gas
        payable(msg.sender).transfer(used * tx.gasprice);
    }

    function _chargeUser(address user, uint256 amount) internal { /* debit deposit */ }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract GasRefundingExecutor {
    uint256 public constant OVERHEAD = 21_000;
    uint256 public maxRefundGas = 1_000_000;          // adjustable
    uint256 public stateGasSurcharge;                 // per call, set for what targets are known to create

    function executeFor(address user, address target, bytes calldata data, uint256 gasLimit) external {
        require(gasLimit <= maxRefundGas, "gas limit too high");
        uint256 start = gasleft();

        (bool ok, ) = target.call{gas: gasLimit}(data);
        require(ok, "call failed");

        uint256 end = gasleft();
        uint256 used = end < start ? start - end : 0; // never underflows
        used += OVERHEAD + stateGasSurcharge;         // state gas the meter cannot see
        if (used > maxRefundGas) used = maxRefundGas; // bound the payout

        uint256 amount = used * tx.gasprice;
        _chargeUser(user, amount);
        (bool sent, ) = payable(msg.sender).call{value: amount}("");
        require(sent, "payment failed");
    }

    function _chargeUser(address user, uint256 amount) internal { /* debit deposit */ }
}
```

The fix bounds the damage: it cannot underflow and the payout is capped. Because `gasleft()` cannot see
reservoir-paid state gas, any gas-metering design has to charge for state creation separately or accept the gap.

## Common pitfalls

- **`unchecked` subtraction of two `gasleft()` readings.** A wrap-around turns a refund into a drain.
- **Checked subtraction.** It does not wrap but reverts, so the operation becomes impossible in that case.
- **Self-billing contracts and paymasters** that charge users by measured gas: account-abstraction entry points,
  relayers repaid per request, gas-refund schemes.
- **Minimum-gas guards** (`require(gasleft() >= minGas)`) meant to make sure a later step, such as a deployment,
  can complete. They cannot see the reservoir.
- **`while (gasleft() > X)` loops.** A refill inside the body lets them run longer than intended.

## Other things to consider

- **Treat a `gasleft()` difference as an estimate of normal gas only.** It never was exact (refunds and the 63/64
  rule already distorted it), and now it misses a whole class of cost.
- **The rule is under discussion.** A proposed spec change (EIPs PR #12581) stops `gasleft()` from rising. That
  fixes problem 1 but not problem 2.
- **Related:** INV-2 (where the refill comes from).
