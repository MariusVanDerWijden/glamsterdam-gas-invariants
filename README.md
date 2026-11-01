# EIP-8037/8038: broken invariants, for auditors

With Glamsterdam we are breaking certain invariants of the EVM. Chainsecurity recently published a [blog post](https://www.chainsecurity.com/blog/glamsterdam-gas/index.html) about this. We have been looking into all available contracts on sourcify to identify contracts that might be broken by these changes.
Today we would like to urge you as auditors to go through your clients code to make sure they are not affected.

Not only the on-chain contracts might be affected, off-chain components can also be affected (e.g. by changes to the receipts).

If you feel like you or one of your clients projects are affected, please contact us ONLY with a working exploit

## Checking your code with an AI agent

Point your coding agent at this repo together with your code, for example: "Check my contracts against the
invariants in glamsterdam-gas-invariants". It will follow [`AGENTS.md`](AGENTS.md):

- map where your code and off-chain services assume gas costs;
- search for the red flags of each invariant;
- estimate the gas each path needs under the new prices;
- test where your toolchain supports the Glamsterdam rules;
- report each issue as checked or inferred, with a suggested fix.

Your code and the results stay with you.

## Gas costs that matter

These values were measured in EELS on the pinned Amsterdam spec. They change if the spec changes.

| Operation | Osaka | Amsterdam |
|---|---|---|
| SSTORE to a new slot (zero to non-zero), cold | 22,100 | 2,100 + 10,000 + **97,920** state gas = 110,020 |
| SSTORE to an existing slot, warm / cold (EIP-8038) | 2,900 / 5,000 | 10,100 / 12,100 |
| New account: value transfer to an empty address, or CREATE | 25,000 | **183,600** state gas plus execution |
| Code deposit | 200 per byte | **1,530** per byte, as state gas |
| EIP-7702 authorization for an empty authority | 25,000 | 235,606 (183,600 of it state gas, charged at runtime) |
| Reservoir (state gas kept apart from `gasleft()`) | – | only when `tx.gas > 2^24` (16,777,216). Below that, all state gas comes out of `gasleft()` |

## Invariants

**Type:**
- **Price (P)**: any change in gas prices would break this;
- **Accounting (A)**: comes from EIP-8037's new way of tracking gas.

**Reservoir:** storage gas kept separate from normal gas. It exists only when a transaction's gas limit is
above 2^24.

| ID | Old assumption | Type | What is different now | Red flags in code | Fix |
|---|---|---|---|---|---|
| [INV-1](invariants/inv-1.md) | If a call runs out of gas, the whole transaction fails | A | With a reservoir, the code after a failed call can still write storage. The transaction finishes and records the failure | After a failed call, the code writes new storage and marks the item done for good, and anyone can pick the gas. `call{gas: X}` does not limit the callee's storage writes | Let failed items be retried, or revert when the call got too little gas |
| [INV-2](invariants/inv-2.md) | Deleting a value set earlier in the same transaction gives no gas back until the end | A | Setting a slot and clearing it in the same transaction gives the gas back at once | Locks or flags set and reset to zero within one transaction, especially around an external call | Use transient storage for locks and flags, or keep their resting value non-zero |
| [INV-3](invariants/inv-3.md) | Gas used by a call that reverts is gone | A | A reverting call hands back the storage gas it used, so the caller has more gas left than expected. A failed contract creation hands back 183,600 | Code that guesses "it ran out of gas" from empty revert data or from the gas left; unchecked CREATE/CREATE2 results | Check the result of CREATE. Do not guess why a call failed from the gas |
| [INV-4](invariants/inv-4.md) | A fixed gas amount is enough for the job | P | Creating storage, accounts and contracts costs 5 to 8 times more. Updating existing storage costs about 7,200 more per write | Hard-coded gas numbers in the contract, its settings, signed messages, bots or relayers. Watch first-time recipients: a new token balance or a new address | Make gas limits adjustable, estimate under the new rules and add a margin |
| [INV-5](invariants/inv-5.md) | Gas left only goes down, so "gas at start minus gas at end" is what the work cost | A | Gas left can go up during a call, and storage gas paid from the reservoir does not show up in it | Code that subtracts two `gasleft()` readings to charge or refund someone | Use checked math, and charge for new storage directly instead of measuring it |
| [INV-6](invariants/inv-6.md) | After a call runs out of gas, the gas left over (1/64) is too little to do anything | P | At the new prices, that leftover can be enough to finish and record a failure | A call with no gas check, followed by code that marks the item done for good. Gas estimates can land in this range | Check there is enough gas before the call, and let failures be retried |
| [INV-7](invariants/inv-7.md) | Gas reserves and stipends sized for today are enough | P | Writing storage after a call costs much more. The 2,300 stipend of `transfer`/`send` cannot create a storage slot | Fixed reserves compared with `gasleft()`; `transfer`/`send` to contracts that write storage; a nonce used up before a call that can fail | Recalculate reserves at the new prices. Do not use up a nonce before a call that can fail |
| [INV-8](invariants/inv-8.md) | A 7702 delegation in a mined transaction was applied and cannot be reused | A | A delegation that runs out of gas is rolled back, but its transaction is still mined. Anyone can reuse the signed delegation | Wallet `setup` or `initialize` functions that anyone can call. Off-chain: treating a mined transaction as proof the delegation worked | Tie setup to the signature. Check the account's code after the transaction |

## Next steps

We are trying to collect feedback in order to have a well-founded discussion on the next ACD about the impact of changing these invariants.
Your input is critical for that. Depending on the response we might decide to slightly change EIP-8037/8038 or not.

Most findings we had reference INV-4 and INV-7, so they should probably be checked first.
Please use the invariants to make sure your code is secure and respond back if not. 