# H-01: WBTC collateral is misvalued because the engine assumes collateral amounts use 18 decimals

## Summary

`DSCEngine` values collateral as if every token amount is denominated with 18 decimals. This works for WETH, but WBTC uses 8 decimals. As a result, WBTC collateral is not normalized correctly before the health factor is calculated.

## Vulnerability Details

The intended behavior is that WETH and WBTC deposits should both be converted into 18-decimal USD values before the user's health factor is checked. A user's ability to mint DSC, redeem collateral, and avoid liquidation depends on this health factor.

The health-factor check starts in `_revert_if_health_factor_is_broken`:

```vyper
@internal
def _revert_if_health_factor_is_broken(user: address):
    user_health_factor: uint256 = self._health_factor(user)
    assert (
        user_health_factor >= MIN_HEALTH_FACTOR
    ), "DSCEngine__BreaksHealthFactor"
```

That calls `_health_factor`, which uses `_get_account_collateral_value`:

```vyper
@internal
@view
def _get_account_collateral_value(user: address) -> uint256:
    total_collateral_value_in_usd: uint256 = 0
    for token: address in COLLATERAL_TOKENS:
        amount: uint256 = self.user_to_token_address_to_amount_deposited[user][
            token
        ]
        total_collateral_value_in_usd += self._get_usd_value(token, amount)
    return total_collateral_value_in_usd
```

The root cause is in `_get_usd_value`:

```vyper
@internal
@view
def _get_usd_value(token: address, amount: uint256) -> uint256:
    price_feed: AggregatorV3Interface = AggregatorV3Interface(
        self.token_address_to_price_feed[token]
    )
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    (
        round_id, price, started_at, updated_at, answered_in_round
    ) = oracle_lib._stale_check_latest_round_data(price_feed.address)
    return (
        (convert(price, uint256) * ADDITIONAL_FEED_PRECISION) * amount
    ) // PRECISION
```

`PRECISION` is `1e18`, so this formula assumes that `amount` is also an 18-decimal token amount. WETH fits that assumption because `1 WETH = 1e18` units. WBTC does not; `1 WBTC = 1e8` satoshi-style units.

Because the code does not read or store the collateral token decimals, WBTC accounting is scaled incorrectly throughout the engine.

## Impact

The engine computes the wrong USD value for WBTC collateral. Since the health factor is derived from this collateral value, WBTC positions can receive incorrect minting limits and liquidation behavior.

This breaks the core accounting path for an in-scope collateral token. The challenge README lists WETH and WBTC as compatible tokens, so WBTC should be handled correctly by the stablecoin engine.

## Proof of Concept

For a simplified WBTC example:

```text
BTC price feed answer = 60,000e8
ADDITIONAL_FEED_PRECISION = 1e10
PRECISION = 1e18
1 WBTC amount = 1e8
```

The engine calculates:

```text
usd_value = (60,000e8 * 1e10 * 1e8) / 1e18
          = 60,000e8
```

But the expected 18-decimal USD value is:

```text
60,000e18
```

So the WBTC collateral value is off by `1e10` because the token amount was never normalized from 8 decimals to 18 decimals.

The same issue affects the complete health-factor path:

```text
deposit WBTC
  -> user_to_token_address_to_amount_deposited stores raw 8-decimal WBTC units
  -> _get_account_collateral_value passes those raw units to _get_usd_value
  -> _get_usd_value divides by 1e18 as if the raw amount had 18 decimals
  -> _calculate_health_factor uses the incorrect USD value
  -> _revert_if_health_factor_is_broken enforces safety using the incorrect health factor
```

## Recommended Mitigation

Normalize collateral amounts using each token's actual decimals before converting to USD.

One approach is to store each collateral token's decimals at deployment:

```diff
+ token_address_to_decimals: public(HashMap[address, uint256])

def __init__(
    token_addresses: address[2],
    price_feed_addresses: address[2],
    dsc_address: address,
):
    DSC = i_decentralized_stable_coin(dsc_address)
    COLLATERAL_TOKENS = token_addresses
    self.token_address_to_price_feed[token_addresses[0]] = price_feed_addresses[0]
    self.token_address_to_price_feed[token_addresses[1]] = price_feed_addresses[1]
+   self.token_address_to_decimals[token_addresses[0]] = staticcall IERC20Detailed(token_addresses[0]).decimals()
+   self.token_address_to_decimals[token_addresses[1]] = staticcall IERC20Detailed(token_addresses[1]).decimals()
```

Then normalize the token amount in `_get_usd_value`:

```diff
- return (
-     (convert(price, uint256) * ADDITIONAL_FEED_PRECISION) * amount
- ) // PRECISION
+ token_decimals: uint256 = self.token_address_to_decimals[token]
+ normalized_amount: uint256 = amount * PRECISION // (10 ** token_decimals)
+ return (
+     (convert(price, uint256) * ADDITIONAL_FEED_PRECISION) * normalized_amount
+ ) // PRECISION
```

The exact Vyper implementation can vary, but the important fix is that WETH, WBTC, and any other supported collateral token must be normalized to a common precision before health-factor accounting.

## Learning Notes

This finding is a reminder that price feed decimals and token decimals are separate concepts. The code correctly tries to convert an 8-decimal oracle price to 18 decimals, but it forgets to normalize the collateral token amount itself.

Relevant concepts: token decimals, oracle precision, health factor accounting, stablecoin collateral valuation.

## Analysis Disclosure

Missed finding, documented after reviewing the challenge results.
