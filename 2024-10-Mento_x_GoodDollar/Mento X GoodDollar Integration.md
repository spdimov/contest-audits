

## Missing slippage protection for updateRatioForReward

### Summary

`updateRatioForReward()` is used when GoodProtocol DAO decides to pay out a reward/grant in G$. It works by decreasing the reserve ratio and minting new tokens. The amount of minted tokens is such that the price of the token remains the same (`price = reserveBalance/(tokenSupply * reserveRatio)`).

Interesting part of this price formula is that it defines a family of curves, and the reserve ratio does not solely define the exact slope of that curve. Instead, the values for `tokenSupply` and `reserveBalance` ultimately effect the slope. This makes it possible to have a dynamic price curve that adjusts to inflation and deflation of a token. You can check [here](https://billyrennekamp.medium.com/converting-between-bancor-and-bonding-curve-price-formulas-9c11309062f5) how the curve would look like with different reserve ratios, but the main idea is that for the same `tokenSupply` and `reserveBalance` having a higher reserve ratio will lead to more linear curve and having a lower reserve ratio would lead to more exponential curve. The more exponential the curve is the more volatile swaps become leading to more volatile prices when swapping tokens.

Another aspect that affects the swaps volatility is the amount of `tokenSupply` and `reserveBalance`. Having higher amounts of both leads to less sensitive price drops when swapping tokens.

Ultimately, the use of `updateRatioForReward()` must be approached with extreme caution, as it causes the bonding curve to become more exponential. The reward minted should be calculated based on the `totalSupply` and `reserveBalance` because having a lower `totalSupply` and correspondingly lower `reserveBalance` will result in a lower reserve ratio for the same amount minted.

The issue with the `updateRatioForReward()` function is that it does not provide slippage protection concerning the new reserve ratio. As previously explained, the reserve ratio is a crucial component of the Bancor formula and is the parameter that affects the curve's exponential behavior.
### Root Cause

No slippage protection in `updateRatioForReward()` in `GoodDollarExchangeProvider`.
https://github.com/sherlock-audit/2024-10-mento-update/blob/098b17fb32d294145a7f000d96917d13db8756cc/mento-core/contracts/goodDollar/GoodDollarExchangeProvider.sol#L195

### Internal pre-conditions

N/A

### External pre-conditions

GoodProtocol DAO calls `mintRewardFromReserveRatio()` which utilizes `updateRatioForReward()` after the tokenSupply has lowered. The decrease in tokenSupply need to happen between the accepting the reward amount by the DAO and actually calling the `mintRewardFromReserveRatio()`.

This does not include direct frontrunning of the reward minting and can be done maliciously/unintended by a whale token holder or by many separate users deciding to sell their tokens. Even if this operation is manually done and token supply before calling the `mintRewardFromReserveRatio()` is checked off-chain there are not any guarantees that it will be the same at the time of executing the transaction.

### Attack Path

As described in the previous sections, changing the reserve ratio to a much lower value than expected—which is undesirable for both the protocol and its users due to its impact on future price volatility—can occur because of changes in the token supply after a certain reward has been approved for minting. This discrepancy can happen unintentionally through the normal flow of users swapping their tokens, or intentionally by a single large holder, or even maliciously by a griefer who knows that a certain amount will be minted and that the ratio will be lowered. The issue is not merely that the reserve ratio will be reduced, but that the extent of this reduction is unbounded and the protocol cannot react to supply changes.

### Impact

This scenario significantly impacts the token's economics, fundamentally altering the bonding curve and increasing the token's price sensitivity. By lowering the reserve ratio and minting new tokens to keep the price the same, the protocol changes how the token's price responds to future trades, introducing potential risks to both the protocol and its users.

Crucially, the impact is directly related to the magnitude of the reserve ratio decrease and the overall token supply at the time of minting. Minting a fixed amount of tokens (X tokens) can result in different degrees of reserve ratio reduction at different moments, depending on the existing total token supply. A smaller total supply means that minting the same number of tokens will lead to a more significant decrease in the reserve ratio, exacerbating price sensitivity and potential volatility.

### PoC

First, I want to show how the delta of the reserve ratio can be mathematically presented:

Initially, the price is calculated via `reserveBalance / (tokenSupply * reserveRatio)`.
Minting new tokens and keeping the price the same can be presented with the following formula:

`reserveBalance / (tokenSupply * reserveRatio) = reserveBalance / ((tokenSupply + reward) * (reserveRatio - delta))

```math 
\frac{1}{\text{tokenSupply} \times \text{reserveRatio}} = \frac{1}{(\text{tokenSupply} + \text{reward}) \times (\text{reserveRatio} - \delta)}
```


```math
\delta (\text{tokenSupply} + \text{reward}) = \text{reward} \times \text{reserveRatio} 
```


```math
\delta = \frac{\text{reward} \times \text{reserveRatio}}{\text{tokenSupply} + \text{reward}}
```


```math
 \boxed{\delta = \frac{\text{reward} \times \text{reserveRatio}}{\text{tokenSupply} + \text{reward}}}
```

It can be observed that the change in the reserve ratio is inversely related to the total token supply. Specifically, a higher token supply results in a smaller change in the reserve ratio because the token supply is in the denominator of the formula for δ\deltaδ. Conversely, a lower token supply leads to a greater decrease in the reserve ratio. This means that when the existing token supply is small, minting a fixed amount of new tokens will have a more significant impact on lowering the reserve ratio, thereby increasing the token's price sensitivity and potential volatility.



Example of the described issue:

At the time of the DAO reward approval we have (taken from the protocol tests):
- Token Supply: 300,000
- Reserve Balance: 60,000 
- Reserve Ratio: 0.2 

The DAO decides to mint 30,000 tokens as a reward.
Using the provided formula to calculate the reserve ratio delta we find it is ~ 0.018 making the expected new ratio = 0.182


Scenario at the Time of Execution:
Due to previous token swaps, the balances have changed by the time `updateRatioForReward()` is executed:
- Token Supply: 200,000 tokens
- Reserve Balance: Approximately 50,000 units (exact value is not used for this calculation)
- Reserve Ratio: 0.2

Using the provided formula to calculate the reserve ratio delta we find the delta for these values would become ~ 0.026 making the new ratio = 0.174

This example serves to illustrate how the absence of slippage protection can significantly affect the reserve ratio. The reserve ratio is one of the most critical parameters in the entire protocol, as it directly impacts the token's economic model, the bonding curve, and the token's price sensitivity.

### Mitigation

Add a slippage parameter when minting tokens as a rewards. This parameter sets the lowest acceptable reserve ratio resulting from the minting process, ensuring that the reserve ratio does not fall below a predefined threshold.