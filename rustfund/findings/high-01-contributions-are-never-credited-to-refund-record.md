Title: Contributions are never credited to the contributor refund record
Impact: High
Likelihood: High
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

RustFund is expected to let contributors recover their contributed SOL when a campaign deadline is reached and the campaign fails to meet its goal.

The `contribute` instruction transfers SOL into the fund account and increases `fund.amount_raised`, but it never increases `contribution.amount`. Since `refund` uses `contribution.amount` as the amount to return, contributors are recorded with a zero refundable balance and cannot recover their SOL through the intended refund path.

```rust
pub fn contribute(ctx: Context<FundContribute>, amount: u64) -> Result<()> {
    let fund = &mut ctx.accounts.fund;
    let contribution = &mut ctx.accounts.contribution;

    if contribution.contributor == Pubkey::default() {
        contribution.contributor = ctx.accounts.contributor.key();
        contribution.fund = fund.key();
@>      contribution.amount = 0;
    }

    system_program::transfer(cpi_context, amount)?;
@>  fund.amount_raised += amount;
    Ok(())
}

pub fn refund(ctx: Context<FundRefund>) -> Result<()> {
@>  let amount = ctx.accounts.contribution.amount;
    // ...
@>  ctx.accounts.contribution.amount = 0;
    Ok(())
}
```

## Risk

**Likelihood**:

* This occurs for every normal contribution because `contribution.amount` is initialized to zero and is never incremented after the SOL transfer.

* The refund path always relies on the stale `contribution.amount` field rather than the contributor's actual transferred lamports.

**Impact**:

* Contributors cannot receive refunds for failed campaigns, directly breaking a core protocol guarantee.

* SOL remains in the fund account and may later be withdrawn by the creator through other flawed withdrawal logic.

## Proof of Concept

```typescript
// 1. Creator creates a fund with goal = 1 SOL.
await program.methods.fundCreate("fund", "desc", new anchor.BN(1_000_000_000)).accounts(...).rpc();

// 2. Creator sets a short deadline.
await program.methods.setDeadline(new anchor.BN(now + 1)).accounts(...).rpc();

// 3. Contributor contributes 0.5 SOL.
await program.methods.contribute(new anchor.BN(500_000_000)).accounts(...).rpc();

// 4. The contribution PDA exists, but amount is still zero.
const contributionAccount = await program.account.contribution.fetch(contributionPda);
expect(contributionAccount.amount.toNumber()).to.equal(0);

// 5. After the deadline, refund sends 0 lamports because refund() uses contribution.amount.
await program.methods.refund().accounts(...).rpc();
```

The contributor sent `500_000_000` lamports, but the program records `0` lamports as refundable. The refund succeeds with a zero-value refund and then sets the already-zero value back to zero.

## Recommended Mitigation

Credit the contributor's refund record whenever a contribution succeeds, and use checked arithmetic for both accounting fields.

```diff
 system_program::transfer(cpi_context, amount)?;
-fund.amount_raised += amount;
+contribution.amount = contribution
+    .amount
+    .checked_add(amount)
+    .ok_or(ErrorCode::CalculationOverflow)?;
+fund.amount_raised = fund
+    .amount_raised
+    .checked_add(amount)
+    .ok_or(ErrorCode::CalculationOverflow)?;
 Ok(())
```
