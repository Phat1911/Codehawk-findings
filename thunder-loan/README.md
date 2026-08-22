# Thunder Loan - Security Review

## Challenge

Thunder Loan is a CodeHawks First Flights challenge about an upgradeable flash loan protocol. Liquidity providers deposit ERC20 tokens and receive AssetTokens, while borrowers can take flash loans that must be repaid with a fee in the same transaction.

Source repository referenced by the challenge README: `https://github.com/Cyfrin/2023-11-Thunder-Loan`

## Scope

The reviewed Solidity contracts are:

- `src/interfaces/IFlashLoanReceiver.sol`
- `src/interfaces/IPoolFactory.sol`
- `src/interfaces/ITSwapPool.sol`
- `src/interfaces/IThunderLoan.sol`
- `src/protocol/AssetToken.sol`
- `src/protocol/OracleUpgradeable.sol`
- `src/protocol/ThunderLoan.sol`
- `src/upgradedProtocol/ThunderLoanUpgraded.sol`

## Context

This review was completed as part of my CodeHawks learning practice. Findings were identified and refined during a guided review of the protocol mechanics, flash loan callback flow, AssetToken accounting, and UUPS upgrade behavior.

This review is not presented as an independent professional audit. It is part of a learning portfolio.

Analysis Type: Mixed - guided independent review + challenge learning

## Findings Summary

| ID | Severity | Title | Status |
| --- | --- | --- | --- |
| H-01 | High | `deposit()` updates the exchange rate with fake fee revenue | Guided independent review |
| H-02 | High | Storage layout corruption after upgrade changes flash loan fee to 100% | Guided independent review |
| H-03 | High | Borrower can repay flash loan via `deposit()` and redeem the minted shares | Guided independent review |
| M-01 | Medium | Disabling an allowed token can lock LP funds in the old AssetToken | Guided independent review |

## Proof Of Concept Tests

A local PoC test file was added at:

- `2023-11-Thunder-Loan/test/unit/ThunderLoanFindingsPoC.t.sol`

The tests could not be compiled in the current local environment because the challenge submodules were empty and dependency installation failed due to a local GitHub SSL certificate issue.
