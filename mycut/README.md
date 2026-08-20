# MyCut - Security Review

## Challenge

MyCut is a CodeHawks First Flights challenge about a contest reward distribution protocol. An owner/admin can create and fund reward pots, authorized users can claim their assigned cut for 90 days, and after the claim period the remaining pool should be split between the manager and users who claimed in time.

Source repository referenced by the challenge README: `https://github.com/Cyfrin/2024-08-MyCut.git`

## Scope

The reviewed Solidity contracts are:

- `src/ContestManager.sol`
- `src/Pot.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Some findings were identified while reviewing the code, and some were refined after comparing against challenge feedback/results.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - post-results learning + independent review

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | `closePot()` can be permanently DoS'd when too many claimants must be paid in one transaction | Post-results learning |
| M-01 | Medium | Manager cut is sent to `ContestManager` instead of the owner/admin and becomes stuck | Post-results learning |
| M-02 | Medium | Duplicate player addresses overwrite assigned rewards | Post-results learning |
| M-03 | Medium | Remaining rewards are divided by all players instead of successful claimants | Independent review |
| L-01 | Low | Integer division leaves rounding dust stuck in `Pot` | Independent review |

## What I Learned

- `msg.sender` changes at each contract-call boundary. When `ContestManager` calls `Pot.closePot()`, the `Pot` sees `ContestManager` as `msg.sender`, not the owner EOA.
- Push-style loops over unbounded arrays can make settlement functions impossible to execute.
- Reward assignment should validate duplicate recipients instead of silently overwriting mapping values.
- The denominator in reward distribution must match the recipient set that actually receives funds.
- Integer division dust is usually a lower-severity accounting issue unless the stuck value can become material.
