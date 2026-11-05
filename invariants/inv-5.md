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
    struct Request { address user; address target; bytes data; uint256 gasLimit; uint256 fee; uint256 nonce; }

    mapping(address => uint256) public nonces;

    // The relayer is paid the fee the user signed, not a gas measurement.
    function executeFor(Request calldata req /*, bytes calldata signature */) external {
        // ...verify the user's signature over req...
        require(nonces[req.user] == req.nonce, "bad nonce");
        nonces[req.user]++;

        (bool ok, ) = req.target.call{gas: req.gasLimit}(req.data);
        require(ok, "call failed");                   // reverts the nonce too: the relayer can retry

        _chargeUser(req.user, req.fee);
        (bool sent, ) = payable(msg.sender).call{value: req.fee}("");
        require(sent, "payment failed");
    }

    function _chargeUser(address user, uint256 amount) internal { /* debit deposit */ }
}
```

Nothing is measured, so nothing can underflow or miss reservoir-paid state gas, and the relayer cannot raise the
charge through `tx.gasprice`. The user and relayer agree on the fee off-chain, with an estimate made under the new
rules.

If you must meter, for example to refund unused prepayment, treat the reading as an estimate and let a limit the user
signed decide the worst case:

```solidity
uint256 end = gasleft();
uint256 used = end < start ? 0 : start - end;         // gasleft() can rise: clamp, do not wrap or revert
uint256 fee = used * tx.gasprice;
if (fee > req.maxFee) fee = req.maxFee;               // the meter misses reservoir-paid state gas
```

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
- **Do not patch the meter with fixed overheads or surcharges** (a constant 21,000, a per-call state-gas
  surcharge). They encode today's gas schedule and break at the next repricing. Charge an agreed fee instead.
- **The rule is under discussion.** A proposed spec change (EIPs PR #12581) stops `gasleft()` from rising. That
  fixes problem 1 but not problem 2.
- **Related:** INV-2 (where the refill comes from).
