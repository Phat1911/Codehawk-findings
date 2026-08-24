# Algo Ssstablecoinsss - Security Review

## Challenge

Algo Ssstablecoinsss is a CodeHawks First Flights challenge about an overcollateralized Vyper stablecoin. Users deposit WETH or WBTC collateral and mint DSC, a USD-pegged ERC20-style stablecoin.

Source repository referenced by the challenge README: `https://github.com/CodeHawks-Contests/ai-algo-ssstablecoinsss`

## Scope

The reviewed Vyper contracts are:

- `src/decentralized_stable_coin.vy`
- `src/dsc_engine.vy`
- `src/oracle_lib.vy`

## Context

This review was completed as part of my CodeHawks learning practice. The accepted issue documented here was added after comparing my submitted findings with the challenge results.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Post-results learning from missed finding

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | WBTC collateral is misvalued because the engine assumes collateral amounts use 18 decimals | Missed finding, post-results learning |

## Proof Of Concept Tests

No local PoC test file was added for this challenge. The finding is documented with the relevant Vyper code path and arithmetic example.
