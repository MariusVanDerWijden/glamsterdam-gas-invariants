# INV-4: A fixed gas amount is no longer enough for the job

**Type:** Price

EIP-8037 makes creating state much more expensive, and EIP-8038 makes reading and updating existing state more
expensive. Any gas amount fixed before the fork may now be too small: in a contract, in its settings, in a signed
message, or in off-chain software.

## Description

The main changes:

| Operation | Before | After |
|---|---|---|
| Write to a new slot (zero to non-zero), cold | 22,100 | 110,020 (97,920 of it state gas) |
| Update an existing slot, warm / cold | 2,900 / 5,000 | 10,100 / 12,100 |
| Create an account by sending ETH to an empty address | 25,000 | 183,600 of state gas plus execution |
| CREATE / CREATE2 to an empty address | 32,000 | 12,000 + 183,600 of state gas |
| Deploy code | 200 per byte | 1,530 per byte |

Unless the transaction's gas limit is above 2^24, all of this comes out of normal gas. So a call given a fixed amount
of gas pays the full price inside that amount.

The most common case is a **first-time recipient**. Sending a token to someone with a zero balance creates their
balance slot, and sending ETH to an empty address creates an account. A transfer that fits in 65,000 gas today can
need over 150,000.

What happens when the gas runs out depends on where the fixed amount lives:

- **In off-chain software** (a relayer, keeper, oracle or hot wallet with a fixed gas limit): the whole transaction
  reverts and has to be resent with more gas.
- **In the contract, its settings or a signed message** (callback gas, bridge minimum destination gas, `safeTxGas`,
  hook budgets): the inner call fails. If the outer contract catches the failure, the result is often permanent: a
  message stored as failed, a callback skipped, rewards booked wrongly.

## Vulnerable example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

interface IConsumer { function onRandom(uint256 requestId, uint256 randomness) external; }

contract RandomnessOracle {
    uint256 public constant CALLBACK_GAS = 100_000;   // sized for pre-Glamsterdam prices

    mapping(uint256 => address) public consumerOf;

    event Fulfilled(uint256 indexed requestId, bool success);

    function fulfill(uint256 requestId, uint256 randomness) external /* onlyOracleNode */ {
        address consumer = consumerOf[requestId];
        delete consumerOf[requestId];                 // the request is used up either way

        // A consumer that writes two new slots now needs over 200,000 gas here.
        (bool ok, ) = consumer.call{gas: CALLBACK_GAS}(
            abi.encodeCall(IConsumer.onRandom, (requestId, randomness))
        );
        emit Fulfilled(requestId, ok);                // a failed callback is final
    }
}
```

## Fixed example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

interface IConsumer { function onRandom(uint256 requestId, uint256 randomness) external; }

contract RandomnessOracle {
    struct Request { address consumer; uint32 callbackGas; }

    address public immutable governance;
    uint32 public maxCallbackGas;                     // a bound, not a budget; governance can change it

    uint256 public nextRequestId;
    mapping(uint256 => Request) public requests;
    mapping(address consumer => mapping(uint256 requestId => uint256)) public undelivered;

    event Fulfilled(uint256 indexed requestId, bool success);

    constructor(address governance_, uint32 maxCallbackGas_) {
        governance = governance_;
        maxCallbackGas = maxCallbackGas_;
    }

    function setMaxCallbackGas(uint32 newMax) external {
        require(msg.sender == governance, "not governance");
        maxCallbackGas = newMax;
    }

    // The consumer knows what its callback does, so it picks the gas (and pays for it).
    function request(uint32 callbackGas) external returns (uint256 requestId) {
        require(callbackGas <= maxCallbackGas, "callback gas too high");
        // ...charge a fee that depends on callbackGas...
        requestId = ++nextRequestId;
        requests[requestId] = Request(msg.sender, callbackGas);
    }

    function fulfill(uint256 requestId, uint256 randomness) external /* onlyOracleNode */ {
        Request memory r = requests[requestId];
        require(r.consumer != address(0), "unknown request");
        delete requests[requestId];                   // fulfilled once: the node cannot re-roll

        (bool ok, ) = r.consumer.call{gas: r.callbackGas}(
            abi.encodeCall(IConsumer.onRandom, (requestId, randomness))
        );
        if (!ok) {
            // Not final: the consumer reads the value later. This writes a new slot (110,020 gas);
            // if the node did not send enough for it, the whole fulfill reverts and is retried.
            undelivered[r.consumer][requestId] = randomness;
        }
        emit Fulfilled(requestId, ok);
    }
}
```

No gas amount is written into the contract. The consumer picks its callback gas, governance can move the bound, and
a callback that runs out of gas, at any price, loses nothing.

## Common pitfalls

- **"No new slot, so we are fine."** Updating existing storage costs about 7,200 more per write. A function that
  updates five slots needs about 36,000 more gas.
- **A slot that was cleared in an earlier transaction is new again.** Writing it costs the full 97,920 of state gas.
- **Fixed limits outside the contract:** relayer and keeper gas limits, oracle reporter limits, exchange
  withdrawal batches, wallet defaults. These do not appear in contract code at all.
- **Gas values that cannot change after signing or sending:** signed meta-transactions, multisig transactions
  signed before the fork, cross-chain messages already in flight with their gas set on the source chain.
- **Estimating with a thin margin.** Estimates made before the fork are wrong after it. Where failures are caught,
  even a fresh estimate can land on a gas at which the transaction "succeeds" without doing its work.
- **Creating and destroying an account in the same transaction** (for example force-sending ETH with a temporary
  contract) still pays the full 183,600.
- **Large deployments.** Contract code at 1,530 gas per byte can push factory deployments and governance payloads
  beyond the per-transaction or block gas limit.

## Other things to consider

- **Make every gas limit adjustable**, with an owner or governance setter and a sane upper bound. Hard-coded
  constants and immutables are what turn a repricing into a migration.
- **Size for the worst common case:** a new holder, a new account, the first write in a period.
- **A gas limit above 2^24 changes the outcome.** The reservoir then pays state gas inside capped calls, so many of
  these failures disappear. Test both sides, but do not rely on senders using such limits.
- **The prices may still change** before the fork. Check against the spec version you test with.
