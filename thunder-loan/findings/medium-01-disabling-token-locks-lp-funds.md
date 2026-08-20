# M-01: Disabling an allowed token can lock LP funds in the old AssetToken

## Summary

When the owner disables an allowed token, `setAllowedToken(token, false)` deletes the token-to-AssetToken mapping. Since `redeem()` requires the token to be allowed, LPs can no longer redeem their shares. Re-enabling the token deploys a new AssetToken, leaving the old AssetToken and its underlying balance disconnected from ThunderLoan.

## Vulnerability Details

The owner can disable an allowed token:

```solidity
function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
    if (allowed) {
        if (address(s_tokenToAssetToken[token]) != address(0)) {
            revert ThunderLoan__AlreadyAllowed();
        }
        string memory name = string.concat("ThunderLoan ", IERC20Metadata(address(token)).name());
        string memory symbol = string.concat("tl", IERC20Metadata(address(token)).symbol());
        AssetToken assetToken = new AssetToken(address(this), token, name, symbol);
        s_tokenToAssetToken[token] = assetToken;
        emit AllowedTokenSet(token, assetToken, allowed);
        return assetToken;
    } else {
        AssetToken assetToken = s_tokenToAssetToken[token];
        delete s_tokenToAssetToken[token];
        emit AllowedTokenSet(token, assetToken, allowed);
        return assetToken;
    }
}
```

After the mapping is deleted, `isAllowedToken(token)` returns false. `redeem()` uses `revertIfNotAllowedToken(token)`, so existing LPs are blocked from redeeming:

```solidity
function redeem(
    IERC20 token,
    uint256 amountOfAssetToken
)
    external
    revertIfZero(amountOfAssetToken)
    revertIfNotAllowedToken(token)
{
    AssetToken assetToken = s_tokenToAssetToken[token];
    ...
}
```

If the owner later calls `setAllowedToken(token, true)`, the protocol deploys a new AssetToken. The old AssetToken still holds the old underlying tokens, but ThunderLoan no longer has a mapping entry that points to it.

## Impact

LP funds can become locked in the old AssetToken after an administrative token disable action. Re-enabling the token does not restore access to the old vault.

This can permanently break redemptions for affected LPs unless a new rescue or migration mechanism is added.

## Proof of Concept

An existing PoC was added in `2023-11-Thunder-Loan/test/unit/ThunderLoanFindingsPoC.t.sol`:

```solidity
function testDisablingTokenLocksLpFunds() public {
    ThunderLoan thunderLoan = _deployThunderLoan();
    thunderLoan.setAllowedToken(tokenA, true);
    AssetToken oldAssetToken = thunderLoan.getAssetFromToken(tokenA);

    address lp = address(1);
    uint256 amount = 1000e18;
    tokenA.mint(lp, amount);

    vm.startPrank(lp);
    tokenA.approve(address(thunderLoan), amount);
    thunderLoan.deposit(tokenA, amount);
    vm.stopPrank();

    assertEq(tokenA.balanceOf(address(oldAssetToken)), amount);
    assertEq(oldAssetToken.balanceOf(lp), amount);

    thunderLoan.setAllowedToken(tokenA, false);

    vm.prank(lp);
    vm.expectRevert();
    thunderLoan.redeem(tokenA, amount);

    thunderLoan.setAllowedToken(tokenA, true);
    AssetToken newAssetToken = thunderLoan.getAssetFromToken(tokenA);

    assertTrue(address(newAssetToken) != address(oldAssetToken));
    assertEq(tokenA.balanceOf(address(oldAssetToken)), amount);
}
```

Reproduction steps:

1. Allow `tokenA`.
2. LP deposits `tokenA` and receives shares from the first AssetToken.
3. Owner disables `tokenA`.
4. LP attempts to redeem and the call reverts.
5. Owner re-enables `tokenA`.
6. ThunderLoan deploys a new AssetToken, while the old AssetToken still holds the LP's underlying tokens.

## Recommended Mitigation

Do not allow disabling a token while active shares or underlying balances remain:

```diff
} else {
    AssetToken assetToken = s_tokenToAssetToken[token];
+   require(assetToken.totalSupply() == 0, "active deposits");
    delete s_tokenToAssetToken[token];
    emit AllowedTokenSet(token, assetToken, allowed);
    return assetToken;
}
```

Alternatively, separate "active for new deposits/flash loans" from "redeemable". Keep `s_tokenToAssetToken[token]` permanently mapped once created, and use a separate boolean to block new activity while still allowing LPs to redeem existing shares.

## Learning Notes

This finding shows why admin disable switches should be designed carefully. Pausing new activity is different from deleting the only route users have to withdraw existing funds.

Relevant concepts: admin controls, withdrawal liveness, token delisting, vault migration.

## Analysis Disclosure

Guided independent review.
