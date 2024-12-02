
## M-01: AMO bot can move the price higher/lower than target $1 which should not be allowed

### Summary

AMO role is to keep the balance of the BOOST token approximately $1. It is done by selling a free minted BOOST when price is over $1 and buying it with USD when price is under $1.
From the contest README:
>Are there any limitations on values set by admins (or other roles) in the codebase, including restrictions on array lengths?
	No, there are no off-chain limitations!  
	There are, however, some hard-coded limitations —— mainly to ensure that even admins can only buyback BOOST below peg and sell it above peg. These are doctsringed.

In V3 implementation AMO rebalance operations has a price limit set for the swap to ensure that when buying BOOST the price of it wont go over `targetSqrtPriceX96` and when selling it to not go under `targetSqrtPriceX96`.

In V2 the AMO rebalance operations does not check the final price of the BOOST. It is only checked that the price is < $1 when buying it and > $1 when selling it. There are not checks to verify that the new price is ~= $1. This creates the following possibilities:

1. Price of BOOST is `$1-X`, AMO calls `unfarmBuyBurn()`, price become up to `$1+X`.
2. Price of BOOST is `$1+X`, AMO calls `mintSellFarm()`, price become up to `$1-X`.

### Root cause

Missing final price check when AMO rebalance the pool.
### Internal pre-conditions

Price of BOOST is under/over $1.
### External pre-conditions

* AMO bot calls the corresponding rebalance function with wrong parameters
	or
* AMO rebalance operation is frontrun in a way that price of BOOST becomes more closer to $1 than the time of initiating the AMO operation

### Attack Path

N/A

### Impact

Inconsistency with specified expected limitations of behavior of the AMO - it is trusted entity but since there are not offchain limitations and it is stated in the README that admins can only buyback BOOST below peg and sell it above peg. This limitation is to ensure that admin actions will always be in favour of the protocol/users and they will only be able to move the price towards $1.
### PoC
https://github.com/sherlock-audit/2024-10-axion/blob/c65e662999d0c79439703fc6713814b4ad023e01/liquidity-amo/contracts/SolidlyV2AMO.sol#L171

The `_mintAndSellBoost()` function is part of `mintSellFarm()`. It includes a check: `if (minUsdAmountOut < toUsdAmount(boostAmount)) minUsdAmountOut = toUsdAmount(boostAmount);`. This ensures that the USD amount from the swap will be higher than the USD equivalent of the BOOST being swapped (i.e., BOOST is valued higher than USD).

The `minUsdAmountOut` ensures that the USD received from the swap is sufficient when the price of BOOST moves from `1+X` to `1-X`. When the price is in the range `[1, 1+X]`, you get more USD per BOOST than the value calculated by `toUsdAmount(boostAmount)`. Even if the price drops into the range `[1, 1-X]`, where less USD is received per BOOST, the extra USD gained in the `[1, 1+X]` range compensates for the lower amount received later. This balance allows the price to be mirrored as it moves between `1+X` and `1-X` which should not be allowed.

```solidity
function _mintAndSellBoost(
        uint256 boostAmount,
        uint256 minUsdAmountOut,
        uint256 deadline
    ) internal override returns (uint256 boostAmountIn, uint256 usdAmountOut) {
        // Mint the specified amount of BOOST tokens
        IMinter(boostMinter).protocolMint(address(this), boostAmount);

        // Approve the transfer of BOOST tokens to the router
        IERC20Upgradeable(boost).approve(router, boostAmount);

        // Define the route to swap BOOST tokens for USD tokens
        ISolidlyRouter.route[] memory routes = new ISolidlyRouter.route[](1);
        routes[0] = ISolidlyRouter.route(boost, usd, true);

        // check to stop AMO from incorrect rebalance
        if (minUsdAmountOut < toUsdAmount(boostAmount)) minUsdAmountOut = toUsdAmount(boostAmount);

        uint256 usdBalanceBefore = balanceOfToken(usd);
        // Execute the swap and store the amounts of tokens involved
        uint256[] memory amounts = ISolidlyRouter(router).swapExactTokensForTokens(
            boostAmount,
            minUsdAmountOut,
            routes,
            address(this),
            deadline
        );
        uint256 usdBalanceAfter = balanceOfToken(usd);
        boostAmountIn = amounts[0];
        usdAmountOut = amounts[1];

        // we check that selling BOOST yields proportionally more USD
        if (usdAmountOut != usdBalanceAfter - usdBalanceBefore)
            revert UsdAmountOutMismatch(usdAmountOut, usdBalanceAfter - usdBalanceBefore);

        if (usdAmountOut < minUsdAmountOut) revert InsufficientOutputAmount(usdAmountOut, minUsdAmountOut);

        emit MintSell(boostAmount, usdAmountOut);
    }
```

The following test shows the scenario where boost price is over $1 (lets say `1+X`) and AMO can make the price up to `1-X`
```solidity
it("should not move the price under $1 when AMO mintSellFarm is called", async function() {
	// --- BOOST price over $1 setup ---
    const usdToBuy = ethers.parseUnits("3000000", 6);
    const minBoostReceive = ethers.parseUnits("990000", 18);
    const routeBuyBoost = [{
      from: usdAddress,
      to: boostAddress,
      stable: true
    }];
    await testUSD.connect(admin).mint(user.address, usdToBuy);
    await testUSD.connect(user).approve(routerAddress, usdToBuy);
    await router.connect(user).swapExactTokensForTokens(
      usdToBuy,
      minBoostReceive,
      routeBuyBoost,
      user.address,
      deadline
    );
    // --- BOOST price over $1 setup ---
    
    console.log("Before balance boost price: %s", await solidlyV2AMO.boostPrice());
    expect(await solidlyV2AMO.boostPrice()).to.be.gt(ethers.parseUnits("1", 6));

    await expect(solidlyV2AMO.connect(amoBot).mintSellFarm(ethers.parseUnits("5500000", 18), 1, 1, 1, deadline)).to.be.emit(solidlyV2AMO, "MintSell");
    console.log("After balance boost price: %s", await solidlyV2AMO.boostPrice());
    // The following wont be true as the price will be < 1
    expect(await solidlyV2AMO.boostPrice()).to.be.approximately(ethers.parseUnits("1", 6), delta);
  });
```
### Mitigation
Add final price of boost checks like how it is done for the public functions:
` if (newBoostPrice < boostLowerPriceSell) revert PriceNotInRange(newBoostPrice);`

## H-01: Public unfarmBuyBurn may not rebalance the pool correctly

### Summary

Public function `unfarmBuyBurn()` is expected to bring back the BOOST price close to $1 in case it is lower than $1. In AMO V3 it is done by removing liquidity and swapping the removed USD for BOOST. Current implementation wont work if the price is in a range with low liquidity.

### Root Cause

Wrong implementation in calculation of the to be removed liquidity when rebalancing a pool which BOOST price is < $1.
https://github.com/sherlock-audit/2024-10-axion/blob/c65e662999d0c79439703fc6713814b4ad023e01/liquidity-amo/contracts/SolidlyV3AMO.sol#L319

totalLiquidity is the amount of "in range" liquidity and it can be much smaller than the whole liquidity spread in the pool.
The whole liquidity is `boostBalance + usdBalance` in case there are not donated amounts of the both tokens. In case there are user donations to the pool, they can additionally prevent the correct pool rebalance, but as this is not profitable and also expensive for an attacker when there is enough liquidity in pool.

Calculated `liquidity` to be removed will be much smaller than the needed in order to move the BOOST price to $1 if the current in range liquidity is not high enough.
```solidity
function _unfarmBuyBurn() internal override returns (uint256 liquidity, uint256 newBoostPrice) {
        uint256 totalLiquidity = ISolidlyV3Pool(pool).liquidity();
        uint256 boostBalance = IERC20Upgradeable(boost).balanceOf(pool);
        uint256 usdBalance = toBoostAmount(IERC20Upgradeable(usd).balanceOf(pool)); // scaled
        if (boostBalance <= usdBalance) revert PriceNotInRange(boostPrice());

        liquidity = (totalLiquidity * (boostBalance - usdBalance)) / (boostBalance + usdBalance);
        liquidity = (liquidity * LIQUIDITY_COEFF) / FACTOR;

        _unfarmBuyBurn(
            liquidity,
            1, // minBoostRemove
            1, // minUsdRemove
            1, // minBoostAmountOut
            block.timestamp + 1 // deadline
        );
```

### Internal pre-conditions

In range liquidity for the current price is low.
### External pre-conditions

N/A
### Attack Path

N/A
### Impact

`unfarmBuyBurn()` wont make the price of BOOST close to 1$.

### PoC

```solidity
it.only("Should execute unfarmBuyBurn successfully when price is above 1", async function() {
        let limitSqrtPriceX96: bigint;
        const zeroForOne = boost2usd;
        if (zeroForOne) {
          limitSqrtPriceX96 = MIN_SQRT_RATIO + BigInt(10);
        } else {
          limitSqrtPriceX96 = MAX_SQRT_RATIO - BigInt(10);
        }
        const boostToBuy = ethers.parseUnits("1200000", 18); //1000000
        await boost.connect(boostMinter).mint(user.address, boostToBuy);
        await boost.connect(user).approve(poolAddress, boostToBuy);
        await pool.connect(user).swap(
          user.address,
          zeroForOne,
          boostToBuy,
          limitSqrtPriceX96,
          1,
          deadline
        );
        
		// Simulate bigger liquidity spread in the pool
        await boost.connect(boostMinter).mint(poolAddress, ethers.parseUnits("10000000", 18));
        await testUSD.mint(poolAddress, ethers.parseUnits("10000000", 6))
        
        // Boost price is lower than 0.9
        expect(await solidlyV3AMO.boostPrice()).to.be.lt(ethers.parseUnits("0.9", 6));
        
        await expect(solidlyV3AMO.connect(amo).unfarmBuyBurn()).to.emit(solidlyV3AMO, "PublicUnfarmBuyBurnExecuted");
        expect(await solidlyV3AMO.boostPrice()).to.be.approximately(ethers.parseUnits(price, 6), 10);
        expect(await boost.balanceOf(amoAddress)).to.be.equal(0);
        expect(await testUSD.balanceOf(amoAddress)).to.be.lt(
          Math.floor(Number(boostToBuy) * (10 ** 6 - Number(usdUsageRatio)) / 10 ** 18));
      });
```

```text
output:
AssertionError: expected 895194 to be close to 1000000 +/- 10
```
### Mitigation

Calculate the needed to be removed liquidity not based current in range liquidity.