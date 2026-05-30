# Fee-on-Transfer Token Accounting in HTLC Contracts

**Class:** Accounting Desynchronisation / Protocol Invariant Violation  
**Ecosystem:** EVM — Solidity / any HTLC or escrow implementation  
**Severity class:** Critical — Permanent Fund Lock  
**Published:** 2026-05  
**Author:** Refious Security

---

## Overview

Hash Time-Locked Contracts (HTLCs) provide a foundational guarantee: deposited capital is either
claimable by the recipient on preimage revelation, or returnable to the initiator on timelock expiry.
This guarantee must be unconditional.

A recurring implementation flaw breaks this guarantee for Fee-on-Transfer (FoT) and deflationary
ERC-20 tokens. The root cause is a false assumption embedded in the accounting model — that
`safeTransferFrom(sender, contract, amount)` results in the contract receiving exactly `amount` tokens.

The ERC-20 standard does not guarantee this. FoT tokens silently deduct a percentage during every
transfer. When HTLC accounting is derived from the user-supplied `amount` parameter rather than
the actual balance delta after transfer, recorded obligations permanently exceed real holdings from
the moment the swap is created.

---

## The False Assumption

```solidity
// Vulnerable pattern — common across HTLC implementations
(uint256 fee, uint256 net) = _calcFee(amount);          // derived from input
IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
_swaps[swapId] = Swap({ amount: net, ... });             // theoretical net stored
```

The fee calculation and swap record are derived from `amount` before the transfer executes.
For standard ERC-20 tokens this is harmless — the contract receives exactly `amount`.
For FoT tokens, the token contract silently burns a percentage during `safeTransferFrom`,
meaning the contract receives materially less than the obligation it records.

This is an **input selection error**, not an arithmetic error. `_calcFee()` is correct in isolation —
it receives the wrong input.

---

## Protocol Invariant Violation

The invariant that must hold at all times in any HTLC or escrow contract:

```
FOR ALL token T:
  sum(_swaps[id].amount for all ACTIVE swaps using T)
  + collectedFees[T]
  <= IERC20(T).balanceOf(address(this))
```

Under standard ERC-20 tokens this invariant holds with equality after each swap creation.
Under FoT tokens it is violated on the **first swap creation** and the violation grows
linearly with each subsequent swap using the same token.

The contract records obligations it cannot fulfil. This framing is significant: the protocol
does not merely fail to support FoT tokens — it **actively corrupts its internal accounting
state** when they are used.

---

## Execution Trace

Scenario: 5% FoT burn rate, `feeBps = 0`, `amount = 100 tokens`.

| Step | Variable | Value | Notes |
|------|----------|-------|-------|
| 1 | `amount` (input) | 100 tokens | User-supplied parameter |
| 2 | `_calcFee(amount)` → `net` | 100 tokens | `feeBps = 0`, net equals amount |
| 3 | FoT burn on inbound transfer | 5 tokens | Token burns 5% during `safeTransferFrom` |
| 4 | Actual received by contract | 95 tokens | True balance delta after transfer |
| 5 | `_swaps[swapId].amount` | 100 tokens | Stored from step 2, not step 4 |
| 6 | Accounting gap | 5 tokens | Contract holds 95, records 100 |
| 7 | `redeem()` transfer attempt | 100 tokens | Tries to send `s.amount` |
| 8 | FoT burn on outbound transfer | 5 tokens | Token burns another 5% |
| 9 | Available after outbound burn | 90.25 tokens | 95 minus 5% |
| 10 | Result | `ERC20InsufficientBalance` revert | Needs 100, holds 90.25 |

The revert is **deterministic and permanent**. The same accounting logic governs `refund()`,
which also attempts to transfer `s.amount`. Both exit paths fail simultaneously.

---

## Compounding Effect

Each swap created with the same FoT token adds a new accounting discrepancy.
The invariant violation grows linearly:

```
After N swaps with FoT burn rate r and protocol feeBps f:

total_recorded(N) = N × amount × (1 - f/10000)
actual_balance(N) = N × amount × (1 - r)
cumulative_shortage(N) = N × amount × (r - f/10000)
```

For `amount = 1000`, `r = 5%`, `f = 0`:

| Swaps | Shortage |
|-------|----------|
| 1 | 50 tokens |
| 5 | 250 tokens |
| 10 | 500 tokens |

Fee withdrawal functions may also fail as the accounting state degrades.
What begins as token incompatibility becomes **systemic protocol accounting corruption**.

---

## Affected Functions

Any function that:
1. Accepts a user-supplied `amount` parameter
2. Derives fee and net calculations from that parameter
3. Executes `safeTransferFrom` after the derivation
4. Stores the derived net as the swap obligation

is vulnerable to this class. This includes `newSwapToken()`, `createSwap()`, `initiate()`,
and equivalent entry points across HTLC implementations.

---

## Fix — Balance-Delta Pattern

Replace amount-based accounting with a balance-before/after delta measurement.
This measures actual tokens received during transfer execution rather than relying on
the theoretical input parameter.

```solidity
// Fixed pattern
uint256 balanceBefore = IERC20(tokenAddr).balanceOf(address(this));
IERC20(tokenAddr).safeTransferFrom(msg.sender, address(this), amount);
uint256 actualReceived = IERC20(tokenAddr).balanceOf(address(this)) - balanceBefore;

(uint256 fee, uint256 net) = _calcFee(actualReceived);  // correct input
_swaps[swapId] = Swap({ amount: net, ... });             // verified balance stored
```

**Properties of this fix:**

| Check | Result |
|-------|--------|
| Fixes accounting desynchronisation | Yes — `swap.amount` always reflects actual holdings |
| Prevents `redeem()` revert | Yes — recorded amount matches available balance |
| Prevents `refund()` revert | Yes — same guarantee applies to initiator exit path |
| Works for standard ERC-20 tokens | Yes — `actualReceived` equals `amount`; behaviour unchanged |
| Works for FoT tokens | Yes — `actualReceived` correctly captures post-fee balance |
| Requires changes to `redeem()` / `refund()` | No — fix is self-contained to swap creation |
| Gas overhead | ~4,200 gas (two `balanceOf` calls) — less than 3.5% of typical tx cost |

---

## Rebasing Token Advisory

The balance-delta fix fully resolves FoT accounting desynchronisation but does not address
**rebasing tokens** (e.g. AMPL). Rebasing tokens adjust holder balances automatically at
protocol-defined intervals. A swap may be created with an accurate recorded amount at time T,
but by redemption time T+n a rebase event may have reduced the contract's actual holdings —
producing the same accounting shortfall via a different mechanism.

Mitigations for rebasing tokens require a separate decision: explicit token blacklisting,
continuous balance verification, or NatSpec documentation of unsupported token classes.

---

## Detection Checklist

When reviewing an HTLC or escrow contract, flag any instance where:

- [ ] Fee or net calculations are derived from a user-supplied `amount` before `transferFrom` executes
- [ ] No balance-before/after delta is measured after `transferFrom`
- [ ] The stored swap obligation is not verified against actual contract balance
- [ ] No FoT token warning exists in NatSpec or documentation
- [ ] No token whitelist or blacklist exists for the swap creation function
- [ ] No admin rescue function exists for locked funds

---

## Broader Class Notes

This vulnerability class recurs across HTLC, escrow, and vault implementations because the
false assumption (`transferFrom` delivers exactly `amount`) is deeply embedded in how ERC-20
accounting is taught and implemented. It is present in reference implementations, tutorials,
and production contracts.

The balance-delta pattern is the well-understood fix and has been standard practice in
AMM and vault implementations for several years. Its adoption in HTLC implementations
remains inconsistent.

Protocols planning mainnet deployment with ERC-20 token support should treat FoT
compatibility as a first-class audit requirement, not an edge case.

---

> Refious Security. *Fee-on-Transfer Token Accounting in HTLC Contracts*. 2026-05.  
> https://github.com/refious-security/research
> https://x.com/RefiousSecurity
