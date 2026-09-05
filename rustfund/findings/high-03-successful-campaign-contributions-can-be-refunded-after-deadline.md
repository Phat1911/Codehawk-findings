Title: Refunds do not require the campaign to have failed its goal
Impact: High
Likelihood: Medium
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

The README defines refunds as available only when a deadline is reached and the campaign goal is not met.

The `refund` instruction checks only whether the deadline has passed. It does not verify that `fund.amount_raised < fund.goal`. Once contribution balances are correctly recorded, contributors can request refunds even from successful campaigns after the deadline, draining funds that should belong to the creator.

This issue is partially masked in the submitted code by the separate bug where `contribution.amount` is never incremented. The missing goal-failure check is still a distinct lifecycle bug in the refund authorization logic.

```rust
pub fn refund(ctx: Context<FundRefund>) -> Result<()> {
    let amount = ctx.accounts.contribution.amount;

@>  if ctx.accounts.fund.deadline != 0
@>      && ctx.accounts.fund.deadline > Clock::get().unwrap().unix_timestamp.try_into().unwrap() {
        return Err(ErrorCode::DeadlineNotReached.into());
    }

@>  // Missing: require!(fund.amount_raised < fund.goal, ...)

    **ctx.accounts.fund.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.fund.to_account_info().lamports()
        .checked_sub(amount)
        .ok_or(ProgramError::InsufficientFunds)?;
}
```

## Risk

**Likelihood**:

* This occurs whenever a successful campaign reaches its deadline and contributors have nonzero recorded contribution balances.

* The refund instruction has no creator approval or campaign failure check.

**Impact**:

* Contributors can reclaim SOL from campaigns that successfully met their goal, violating the creator withdrawal guarantee.

* Refunded amounts are not subtracted from `fund.amount_raised`, so later creator withdrawals can fail or attempt to withdraw more lamports than are actually available.

## Proof of Concept

```typescript
// Assumes the contribution accounting bug is fixed so contribution.amount is credited.

// 1. Creator creates a fund with goal = 1 SOL and sets a short deadline.
await program.methods.fundCreate("fund", "desc", new anchor.BN(1_000_000_000)).accounts(...).rpc();
await program.methods.setDeadline(new anchor.BN(now + 1)).accounts(...).rpc();

// 2. Contributor contributes 1 SOL, so the campaign is successful.
await program.methods.contribute(new anchor.BN(1_000_000_000)).accounts(...).rpc();

// 3. After the deadline, the contributor calls refund().
await program.methods.refund().accounts({
  fund: fundPda,
  contribution: contributionPda,
  contributor: contributor.publicKey,
  systemProgram: anchor.web3.SystemProgram.programId,
}).rpc();
```

The refund succeeds because the only checked condition is deadline timing. The program never rejects the refund even though `amount_raised >= goal`.

## Recommended Mitigation

Require the campaign to have failed before allowing refunds, and keep aggregate accounting in sync when a refund succeeds.

```diff
 pub fn refund(ctx: Context<FundRefund>) -> Result<()> {
+    let fund = &mut ctx.accounts.fund;
     let amount = ctx.accounts.contribution.amount;

-    if ctx.accounts.fund.deadline != 0 && ctx.accounts.fund.deadline > Clock::get().unwrap().unix_timestamp.try_into().unwrap() {
+    if fund.deadline != 0 && fund.deadline > Clock::get().unwrap().unix_timestamp.try_into().unwrap() {
         return Err(ErrorCode::DeadlineNotReached.into());
     }
+    require!(fund.amount_raised < fund.goal, ErrorCode::CampaignSuccessful);

     // transfer refund...
+    fund.amount_raised = fund
+        .amount_raised
+        .checked_sub(amount)
+        .ok_or(ErrorCode::CalculationOverflow)?;
     ctx.accounts.contribution.amount = 0;
     Ok(())
 }
```

Add a dedicated error:

```diff
 pub enum ErrorCode {
+    #[msg("Campaign met its funding goal")]
+    CampaignSuccessful,
 }
```
