# INV-8: A 7702 delegation in a mined transaction may not have been applied, and can be reused

**Type:** Accounting

Under EIP-8037, an EIP-7702 delegation transaction with too little gas can be mined with status 0 while its
delegation is rolled back. The signed authorization is then public but still valid, so anyone can reuse it. If the
delegate contract has an unprotected `initialize`, an attacker can take over the account.

## Description

An EIP-7702 authorization is a signed tuple (chain id, delegate address, nonce) that lets an EOA run a delegate
contract's code. It is meant to be single-use: once applied, the account's nonce moves past the signed nonce.

**Before Glamsterdam.** The cost of each authorization was part of the transaction's intrinsic gas. A transaction
that could not pay it was invalid and never mined. So "the transaction is mined" meant "the delegation was applied".

**Under EIP-8037.** Part of the cost depends on the account's current state: creating the account if it is empty
costs 183,600 of state gas. That part is charged when the authorization is processed, not up front. An empty account
now costs 235,606 per authorization, against 25,000 before.

A transaction budgeted at the old cost is accepted, runs out of gas while applying the authorization, and is mined
with status 0. The out-of-gas rolls back the delegation *and* the account's nonce bump. The sender's nonce still
advances.

The signed authorization is now visible in a block and still valid. Anyone can put it in a type-4 transaction of
their own. If the delegate's setup function does not check who is calling, the attacker can apply the delegation
and call `initialize(attackerOwner)` in the same transaction.

Front-running an authorization was possible before, in the public mempool. What is new is that a *failed*
transaction publishes a still-valid authorization at a gas that used to work. Sending it privately does not help.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

// Delegate code that EOAs point to with EIP-7702.
contract SmartAccount {
    address public owner;

    // Anyone can call this once, on any account that delegates here.
    function initialize(address newOwner) external {
        require(owner == address(0), "already initialized");
        owner = newOwner;                             // first caller wins
    }

    function execute(address to, uint256 value, bytes calldata data) external {
        require(msg.sender == owner, "not owner");
        (bool ok, ) = to.call{value: value}(data);
        require(ok, "call failed");
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract SmartAccount {
    address public owner;

    // Under 7702, address(this) is the EOA. Only its key can approve the owner.
    function initialize(address newOwner, bytes calldata signature) external {
        require(owner == address(0), "already initialized");
        bytes32 digest = keccak256(abi.encode(block.chainid, address(this), newOwner, "SmartAccount.initialize"));
        require(ECDSA.recover(digest, signature) == address(this), "not signed by account");
        owner = newOwner;
    }

    // Simpler alternative when the EOA sends the setup itself:
    // require(msg.sender == address(this), "only self");

    function execute(address to, uint256 value, bytes calldata data) external {
        require(msg.sender == owner || msg.sender == address(this), "not owner");
        (bool ok, ) = to.call{value: value}(data);
        require(ok, "call failed");
    }
}
```

Off-chain, after a delegation transaction is mined:

- check the account's code (`0xef0100 || delegate`) and nonce;
- do not rely only on the receipt or its status.

## Common pitfalls

- **Unprotected `setup` or `initialize` on delegate contracts.** Generic singletons that were never meant to be
  delegated to directly are the worst case: their setup is guarded only by "not yet initialized".
- **Treating a mined type-4 transaction as proof the delegation worked.** Wallets, relayers and indexers must read
  the account's resulting code.
- **Budgeting the old 25,000 gas per authorization** in relayers and bundlers. An empty account now costs 235,606.
- **Assuming a broadcast authorization is single-use.** It is single-use only once it has actually been applied.
- **Authorizations signed with chain id 0** are valid on every chain. That is not new, but it widens where a leaked
  authorization can be reused.

## Other things to consider

- **Bind initialization to the account's own signature or to self-calls,** and include the chain id and the account
  address in what is signed.
- **If a delegation transaction fails,** treat the authorization as exposed. Revoke it by sending a transaction that
  advances the account's nonce, or delegate again to a safe contract.
- **A spec change has been suggested** (charging the worst-case authorization cost up front, so an underfunded
  transaction is rejected as before). It is not adopted. Design as if the current behaviour stays.
