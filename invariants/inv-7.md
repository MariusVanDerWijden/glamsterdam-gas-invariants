# INV-7: Gas reserves and stipends sized for today are no longer enough

**Type:** Price

Contracts keep back a fixed amount of gas for the work after a call, or pass a fixed stipend such as the 2,300 of
`transfer`/`send`. When that later work writes storage, it now costs far more than the reserve or stipend covers.

## Description

Three related patterns are affected.

- **Reserves for work after a call.** A contract forwards `gasleft() - RESERVE` to a call and keeps `RESERVE` to
  record the result. Recording a result often writes a new slot: 22,100 before, 110,020 after. A reserve of 40,000
  that was comfortable is now far too small. The contract runs out of gas after the call. Depending on who called
  it, that can leave the result unrecorded with no way to retry.
- **Minimum-gas checks.** `require(gasleft() >= request.gas + RESERVE)` was meant to make sure the call gets its gas
  and the bookkeeping still fits. Sized for old prices, it lets through transactions that then fail halfway. A nonce
  is often already used up, so the user has to sign again.
- **Stipends.** `transfer` and `send` pass 2,300 gas. A receiving contract whose `receive` writes a new slot now needs
  over 110,000. The transfer fails, which may revert the payment or, if the failure is caught, leave the funds
  unaccounted for.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Forwarder {
    struct Request { address from; address to; uint256 gas; uint256 nonce; bytes data; }

    uint256 public constant RESERVE = 40_000;         // sized for pre-Glamsterdam prices
    mapping(address => uint256) public nonces;
    mapping(bytes32 => bool) public executed;

    function execute(Request calldata req /*, bytes calldata signature */) external {
        // ...verify signature...
        require(nonces[req.from] == req.nonce, "bad nonce");
        require(gasleft() >= req.gas + RESERVE, "not enough gas");

        nonces[req.from]++;                           // used up even if the call below fails

        (bool ok, ) = req.to.call{gas: req.gas}(abi.encodePacked(req.data, req.from));

        executed[keccak256(abi.encode(req))] = ok;    // new slot: 110,020, not covered by RESERVE
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Forwarder {
    struct Request { address from; address to; uint256 gas; uint256 nonce; bytes data; }

    address public immutable admin;
    uint256 public callGasMargin;                     // the CALL's own cost (account access, memory); adjustable
    mapping(address => uint256) public nonces;

    event Executed(address indexed from, uint256 nonce, bool success);

    constructor(address admin_, uint256 callGasMargin_) {
        admin = admin_;
        callGasMargin = callGasMargin_;
    }

    function setCallGasMargin(uint256 margin) external {
        require(msg.sender == admin, "not admin");
        callGasMargin = margin;
    }

    function execute(Request calldata req /*, bytes calldata signature */) external {
        // ...verify signature...
        require(nonces[req.from] == req.nonce, "bad nonce");
        nonces[req.from]++;

        // Do all other work first, so no gas is spent between the check and the call.
        bytes memory data = abi.encodePacked(req.data, req.from);
        uint256 margin = callGasMargin;

        // EIP-150: the callee gets at most 63/64 of the gas left at the call. Make sure it gets the
        // gas the user signed for. A failure is then the request's own, and using up the nonce is fine.
        require(gasleft() >= req.gas * 64 / 63 + margin, "not enough gas");
        (bool ok, ) = req.to.call{gas: req.gas}(data);
        emit Executed(req.from, req.nonce, ok);       // an event, not a new slot: nothing to reserve for
    }
}
```

There is no reserve for the work after the call. If that work runs out of gas, the whole transaction reverts,
nonce included, and the relayer retries. A reserve only matters when running out after the call leaves something
permanent behind. The one gas amount the contract needs, the margin for the CALL itself, is a setting, because
EIP-8038 reprices account access.

For ETH payments, avoid the stipend:

```solidity
(bool sent, ) = payable(recipient).call{value: amount}("");
if (!sent) owed[recipient] += amount;                 // let the recipient pull it instead
```

## Common pitfalls

- **Reserves that cover only an event.** Check whether the post-call path writes a slot that may be new: receipts,
  "failed" markers, counters that start at zero.
- **Checking `gasleft() >= gas + reserve` without the 63/64 factor.** The callee gets less than you think.
- **`transfer`/`send` to contracts:** smart wallets, splitters and vaults with logic in `receive`.
- **One-shot markers before a call that can fail:** nonces, "claimed" flags, "executed" flags. Once used up, the user
  has to sign or vote again. This is safe only when the call is known to get the gas that was signed for.
- **Constants and immutables.** A reserve that cannot be changed turns a repricing into a migration.

## Other things to consider

- **Size reserves for the first write,** not the usual one. A slot that is zero when the transaction starts costs
  the full state gas. Better still, remove the need for one: record outcomes in events, or make the work after the
  call revert-safe.
- **Prefer events over storage** for recording outcomes that no on-chain code reads.
- **Off-chain parts carry the same assumption:** relayers that add a fixed amount on top of the request gas, and
  wallets with a fixed per-call allowance.
- **Related:** INV-4 (fixed gas amounts in general) and INV-6 (the 1/64 kept back).
