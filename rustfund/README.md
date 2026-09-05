# RustFund - Security Review

## Challenge

RustFund is a CodeHawks First Flights challenge about a Solana crowdfunding program. Creators can create fundraising campaigns with a name, description, goal, and deadline, while contributors can send SOL to campaigns and request refunds when campaigns fail.

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/2025-03-rustfund`

## Scope

The reviewed Rust/Anchor program is:

- `programs/rustfund/src/lib.rs`

## Context

This review was completed as part of my CodeHawks learning practice while studying Solana, Anchor account constraints, PDAs, lamport transfers, and crowdfunding lifecycle rules.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - guided independent review + challenge learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | Contributions are never credited to the contributor refund record | Guided independent review |
| H-02 | High | Creators can withdraw funds from unsuccessful campaigns | Guided independent review |
| H-03 | High | Refunds do not require the campaign to have failed its goal | Guided independent review |
| H-04 | High | Creators can withdraw campaign funds before the deadline is reached | Guided independent review |
| M-01 | Medium | Creator can repeatedly change a supposedly fixed deadline | Guided independent review |
| M-02 | Medium | Withdrawals leave stale `amount_raised` and block later withdrawals | Guided independent review |
