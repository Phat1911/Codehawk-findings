Title: Creators can withdraw funds from unsuccessful campaigns
Impact: High
Likelihood: High
Scope: programs/rustfund/src/lib.rs

# Root + Impact

## Description

RustFund's README states that creators can withdraw raised funds after successful campaigns, while contributors can request refunds when the campaign fails to meet its goal after the deadline.

The `withdraw` instruction only checks that the signer is the fund creator through the account constraint. It does not verify that the campaign has met its funding goal, and it does not require the campaign to be in a successful state. A creator can therefore withdraw contributor SOL even when the campaign raised less than the goal.

```rust
pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
@>  let amount = ctx.accounts.fund.amount_raised;

@>  **ctx.accounts.fund.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.fund.to_account_info().lamports()
        .checked_sub(amount)
        .ok_or(ProgramError::InsufficientFunds)?;

@>  **ctx.accounts.creator.to_account_info().try_borrow_mut_lamports()? =
        ctx.accounts.creator.to_account_info().lamports()
        .checked_add(amount)
        .ok_or(ErrorCode::CalculationOverflow)?;

    Ok(())
}

#[derive(Accounts)]
pub struct FundWithdraw<'info> {
@>  #[account(mut, seeds = [fund.name.as_bytes(), creator.key().as_ref()], bump,has_one = creator)]
    pub fund: Account<'info, Fund>,
    #[account(mut)]
    pub creator: Signer<'info>,
}
```

## Risk

**Likelihood**:

* Any campaign creator can call `withdraw` after at least one contribution because no success condition is checked.

* The vulnerable path does not require timing precision or special account setup beyond being the fund creator.

**Impact**:

* Contributors can lose SOL to creators even when the campaign failed to meet its funding target.

* The refund mechanism is bypassed because the fund account's lamports are removed before contributors can recover them.

## Proof of Concept

```typescript
// 1. Creator creates a fund with goal = 1 SOL.
await program.methods.fundCreate("fund", "desc", new anchor.BN(1_000_000_000)).accounts(...).rpc();

// 2. A contributor contributes only 0.5 SOL, so the campaign is unsuccessful.
await program.methods.contribute(new anchor.BN(500_000_000)).accounts(...).rpc();

// 3. Creator calls withdraw even though amountRaised < goal.
await program.methods.withdraw().accounts({
  fund: fundPda,
  creator: creator.publicKey,
  systemProgram: anchor.web3.SystemProgram.programId,
}).rpc();
```

The call succeeds because `withdraw` never checks `fund.amount_raised >= fund.goal`. The creator receives the contributed lamports from an unsuccessful campaign.

## Recommended Mitigation

Require campaign success before allowing withdrawal. The exact timing rule depends on the intended lifecycle, but at minimum the campaign should have met its goal.

```diff
 pub fn withdraw(ctx: Context<FundWithdraw>) -> Result<()> {
+    let fund = &ctx.accounts.fund;
+    require!(fund.amount_raised >= fund.goal, ErrorCode::CampaignNotSuccessful);
+
-    let amount = ctx.accounts.fund.amount_raised;
+    let amount = fund.amount_raised;
     // transfer lamports...
     Ok(())
 }
```

Add a dedicated error:

```diff
 pub enum ErrorCode {
+    #[msg("Campaign has not met its funding goal")]
+    CampaignNotSuccessful,
 }
```
