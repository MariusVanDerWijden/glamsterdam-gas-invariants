# Instructions for agents

A developer has pointed you at this repo to check *their own* code for the Glamsterdam gas changes (EIP-8037 and
EIP-8038). This file is the procedure. The invariants are described in `README.md` and in `invariants/inv-1.md` to
`invariants/inv-8.md`.

## Ground rules

1. **Read first.** Read `README.md` and every file in `invariants/` before you look at the user's code. The red flags
   and fixes there are what you check against.
2. **Source code only, no reverse engineering.** Use this repo only on contracts the user owns or that are open
   source. Never decompile, disassemble or otherwise reverse engineer bytecode, even when asked.
   - If the user has the source code available, assume they are authorized to look for issues in it.
   - If all that is available is a deployed address or bytecode, stop. Ask for the source code instead.
3. **The user's code stays with the user.** Do not send it, or anything you find, to any outside service, tracker or
   person.
   - Report to the user only. Whether to contact anyone (see "Next steps" in `README.md`) is their decision.
   - That contact is only for a working exploit.
4. **Say what you checked and what you inferred.** "Checked" means a test or simulation under Glamsterdam rules
   showed it. "Inferred" means you worked it out from the code and the gas table. Never present an inferred issue as
   confirmed.
5. **Do not change the user's code without asking.** Propose fixes and apply them only when the user agrees.
6. **Do not edit this repo** while auditing.
7. **The numbers may change.** The gas costs in `README.md` come from one version of the spec. If the user's toolchain
   implements a different version, use its numbers and say so.

## Step 1: Map what can be affected

Gas assumptions live in more places than the contracts. List:

- **Contracts:** especially ones that call other contracts, catch failures, meter gas, deploy contracts or serve as
  EIP-7702 delegates.
- **Off-chain senders:** relayers, keepers, bots, oracle reporters, deploy scripts, backend wallets, front ends. Look
  for fixed gas limits, fixed margins and gas estimation code.
- **Signed or stored gas values:** meta-transactions, multisig transactions, cross-chain messages, per-request
  callback limits, config set by governance.

Ask the user for anything outside the repo you were given: off-chain services, deployed settings, messages already
in flight.

## Step 2: Search for red flags

Run these searches (ripgrep syntax) as a starting point. A hit is not a bug; it is a place to look at in Step 3. Then
read the code around every external call, even without a hit. The patterns miss renamed variables and inline
assembly.

| Invariant | Search for |
|---|---|
| INV-1, INV-6 (failure branches, starved calls) | `\.call\{`, `\.call\(`, `\btry\b`, `catch`, `raw_call`, `delegatecall`, `assembly.*call\(` |
| INV-2 (slot set and cleared in one tx) | `delete `, `= 0;` near an external call, locks or flags whose idle value is 0, `approve\(.*, 0\)` |
| INV-3 (handed-back gas, unchecked CREATE) | `reason\.length == 0`, `returndatasize\(\)`, `gasleft\(\).*/ *63`, `create2?\(`, `pop\(create`, `new [A-Z]\w*\{salt` |
| INV-4 (fixed gas amounts) | `gas:`, `\{gas:`, `[Gg]asLimit`, `callbackGas`, `minDstGas`, `safeTxGas`, `GAS_` constants. Off-chain: `gasLimit`, `gas:`, `gas=`, `maxGas` in JS/TS/Python/Go/Rust and config files |
| INV-5 (gas metering) | two `gasleft\(\)` readings in one function, `unchecked` near `gasleft`, `gasUsed`, `startGas`, `gasStart` |
| INV-7 (reserves, stipends) | `RESERVE`, `gasleft\(\) *>=?`, `\.transfer\(`, `\.send\(`, nonce or "executed" writes before a call |
| INV-8 (7702) | `initialize\(`, `setup\(`, `0xef0100`. Off-chain: `authorizationList`, type-4 or `0x04` transactions, `signAuthorization` |

## Step 3: Triage each hit

For each place, answer these questions in order and write the answers down.

1. **What runs out of gas?** Is it a whole transaction (an off-chain gas limit) or one inner call (a `gas:` argument,
   a signed limit, a reserve)?
2. **Does the path create state?** New state costs about 88,000 more than today.
   - Write to a slot that may be zero: a first-time token recipient's balance, a new mapping entry, a counter starting
     at 0, a slot cleared in an earlier transaction.
   - Send ETH to an address that may be empty.
   - Deploy a contract.
3. **How many existing slots does it update?** Each costs about 7,200 more. Paths with no new slot can still break.
4. **Who picks the gas?** Anyone (a public function, a permissionless relayer), a trusted operator, or a value fixed
   in the past (signed, stored, immutable)?
5. **What happens on failure?**
   - The transaction reverts and can be retried. This is a liveness problem.
   - A failure is recorded for good, a nonce or flag is used up, funds are booked wrongly or locked. This is a
     correctness problem.
6. **Can the gas amount be changed** without a contract upgrade?

Then match the place to the invariant it fits and check it against that file's "Common pitfalls".

**Priority:** a permanent wrong outcome that anyone can trigger comes first. A liveness issue fixed by changing a
setting comes last.

## Step 4: Estimate the cost under the new prices

Work out the gas the path needs under Glamsterdam and compare it with the gas it gets. Use the table in `README.md`.

| Operation | Before | After |
|---|---|---|
| New slot, cold | 22,100 | 110,020 |
| Existing slot update, warm / cold | 2,900 / 5,000 | 10,100 / 12,100 |
| New account | 25,000 | about 183,600 plus execution |
| Code | 200 per byte | 1,530 per byte |

For example, a callback limited to 100,000 that writes two new slots needs at least 2 × 110,020 = 220,040 for those
writes alone.

Remember the 63/64 rule: a call gets at most 63/64 of the gas left at that point. For INV-6, the caller keeps 1/64 of
what it forwarded. Check whether that is enough to finish the work after the call.

## Step 5: Verify

Confirm the issue only if the user's toolchain can execute Glamsterdam rules (an EVM or test framework that
implements EIP-8037 and EIP-8038, or the Ethereum execution specs). If it cannot, mark the issue "inferred" and say
what test would confirm it.

When you can test:

- **Sweep the gas, do not test one value.** Problems from INV-1, INV-3 and INV-6 appear in a *range* of gas. Step from
  the lowest gas that does not revert up to well above what the call needs, and record the outcome at each step: did
  the inner work happen, and was anything recorded for good?
- **Test both sides of a 2^24 transaction gas limit.** Above it, a state-gas reservoir exists and outcomes change.
- **Test first-time and existing recipients separately:** a zero token balance, an empty ETH address.
- **For INV-2 and INV-5**, assert that `gasleft()` differences are never negative and that metered amounts match
  what was really spent.
- **For INV-8**, send an underfunded delegation transaction to an empty account. Check that it is mined with status 0
  and that the account has no code. Then check whether another sender can reuse the authorization and call setup.
- **Compare with the current rules (Osaka)** where you can, so that you report only what the fork changes.

## Step 6: Report

Give the user one table, ordered by priority:

| Location | Invariant | Who can trigger | What goes wrong | Gas needed vs available | Checked or inferred | Suggested fix |
|---|---|---|---|---|---|---|

After the table:

- **What you did not cover:** code you could not see, off-chain services, deployed settings, messages in flight.
- **For each issue, the fix** from the matching `invariants/inv-N.md`, adapted to the user's code: adjustable gas
  limits, checks before the call, retryable failures, transient storage, checked CREATE results, signed
  initialization. Point out where a fix needs a migration because the gas value is immutable or already signed.
- **Anything that depends on spec changes still under discussion** (INV-1, INV-2, INV-3, INV-5). Recommend a fix that
  works either way.

If you found nothing, say what you searched and which invariants you could not check, rather than declaring the code
safe.
