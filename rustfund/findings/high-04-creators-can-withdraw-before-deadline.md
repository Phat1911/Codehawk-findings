Title: Creators can withdraw campaign funds before the deadline is reached
Impact: High
Likelihood: High
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

RustFund's campaign lifecycle is deadline based: contributors can contribute to active campaigns, and refunds are available when the deadline is reached and the goal is not met. In that model, withdrawal should only happen after the campaign has reached a final state.

The `withdraw` instruction has no deadline check. The creator can call `withdraw` at any time, including while the campaign is still active. This lets the creator remove SOL before contributors know the final campaign outcome.

```rust
pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
@>  let amount = ctx.accounts.fund.amount_raised;

@>  // Missing: require deadline to be reached or campaign to be finalized

    **ctx.accounts.fund.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.fund.to_account_info().lamports()
        .checked_sub(amount)
        .ok_or(ProgramError::InsufficientFunds)?;

    **ctx.accounts.creator.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.creator.to_account_info().lamports()
        .checked_add(amount)
        .ok_or(ErrorCode::CalculationOverflow)?;

    Ok(())
}
```

## Risk

**Likelihood**:

* Any creator can call `withdraw` during an active campaign because the instruction only requires the creator signer.

* The program has no campaign finalization state and no deadline condition in the withdrawal path.

**Impact**:

* Contributor SOL can be removed from the fund account before refund eligibility is determined.

* Active campaigns no longer have their raised SOL escrowed in the program, breaking contributor expectations and the refund lifecycle.

## Proof of Concept

```typescript
// 1. Creator creates a fund with a future deadline.
await program.methods.fundCreate("fund", "desc", new anchor.BN(1_000_000_000)).accounts(...).rpc();
await program.methods.setDeadline(new anchor.BN(now + 7 * 24 * 60 * 60)).accounts(...).rpc();

// 2. Contributor contributes while the campaign is active.
await program.methods.contribute(new anchor.BN(500_000_000)).accounts(...).rpc();

// 3. Creator immediately withdraws before the deadline is reached.
await program.methods.withdraw().accounts({
  fund: fundPda,
  creator: creator.publicKey,
  systemProgram: anchor.web3.SystemProgram.programId,
}).rpc();
```

The withdrawal succeeds even though the deadline is still in the future. The program never checks `fund.deadline <= current_time`.

## Recommended Mitigation

Require the campaign to be finalized before creator withdrawal. For a deadline-based campaign, this means checking that the deadline has been set and reached, then separately checking that the campaign succeeded.

```diff
 pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
+    let fund = &ctx.accounts.fund;
+    let now: u64 = Clock::get()?
+        .unix_timestamp
+        .try_into()
+        .map_err(|_| ErrorCode::CalculationOverflow)?;
+    require!(fund.deadline != 0 && fund.deadline <= now, ErrorCode::DeadlineNotReached);
+    require!(fund.amount_raised >= fund.goal, ErrorCode::CampaignNotSuccessful);
+
-    let amount = ctx.accounts.fund.amount_raised;
+    let amount = fund.amount_raised;
     // transfer lamports...
     Ok(())
 }
```

Add a dedicated campaign success error:

```diff
 pub enum ErrorCode {
+    #[msg("Campaign has not met its funding goal")]
+    CampaignNotSuccessful,
 }
```
