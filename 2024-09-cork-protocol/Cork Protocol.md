
# H-01: Wrong order of parameters when using MathHelper.calculatePrecentageFee()

# H-02: pair.getReserves() return values are not sorted and can lead to wrong calculations

## Summary

When calling the `getReserves()` function in a Uniswap V2 pair contract, it’s essential to also check the `token0` and `token1` of the pair. The reserves returned by `getReserves()` correspond to `token0` and `token1` and are not sorted by token address. Therefore, you cannot directly assign the returned reserve values to specific tokens without first verifying which token corresponds to `token0` and `token1`.

## Vulnerability Detail

The current implementation fails to properly check and assign values for RA and CT tokens when querying reserves from a Uniswap V2 pair contract. In cases where `token0` is RA and `token1` is CT, there is no issue. However, since the token order in a pair is determined by lexicographic sorting and is not guaranteed, the reserves may be incorrectly assigned. To avoid this, the contract must always verify whether `token0` or `token1` corresponds to RA or CT and assign reserves accordingly. Without this check, the protocol risks using incorrect reserve values, leading to potential errors in calculations and swaps.

All places where this vulnerability arises can be checked in the Code Snippet section of the report.
## Impact

This issue can lead to a completely unusable protocol due to incorrect calculations based on misassigned reserves. Since many core functions rely on the correct pair reserves, failing to properly check and assign the RA and CT reserves could result in faulty swaps, liquidity provisions, and other operations, rendering the protocol inoperable. Combining this with the high likelihood of this happening (50% chance for every RA:CT pair), the whole severity is High.

## Code Snippet

https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L147
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L425
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L438
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L498

## Tool used

Manual Review

## Recommendation

Do the needed checks to correctly assign the reserve values like how it is done in `FlashSwapRouter.getReservesSorted()`.


# H-03: VaultLib.calculateTotalRaAndCtBalanceWithReserve() is used with wrong parameter

## Summary

`__calculateTotalRaAndCtBalanceWithReserve()` expects to get the total RA and CT reserves in a pair as well as the total LP supply to calculate the `raPerLv`, `ctPerLv`, `raPerLp`, `ctPerLp`, `totalRa`, `ammCtBalance`. In the both places where this function is used, the provided `lpSupply` value does not represent the total LP supply.

## Vulnerability Detail

```solidity
function __calculateTotalRaAndCtBalanceWithReserve(
        State storage self,
        uint256 raReserve,
        uint256 ctReserve,
        uint256 lpSupply
    )
        internal
        view
        returns (
            uint256 totalRa,
            uint256 ammCtBalance,
            uint256 raPerLv,
            uint256 ctPerLv,
            uint256 raPerLp,
            uint256 ctPerLp
        )
    {
        (raPerLv, ctPerLv, raPerLp, ctPerLp, totalRa, ammCtBalance) = MathHelper.calculateLvValueFromUniV2Lp(
            lpSupply, self.vault.config.lpBalance, raReserve, ctReserve, Asset(self.vault.lv._address).totalSupply()
        );
    }
```

In `VaultLib.__calculateCtBalanceWithRate()` it used as
```solidity
(,, raPerLv, ctPerLv, raPerLp, ctPerLp) = __calculateTotalRaAndCtBalanceWithReserve(
            self, raReserve, ctReserve, flashSwapRouter.getLvReserve(self.info.toId(), dsId)
        );
```

In `VaultLib.__calculateTotalRaAndCtBalance()` it is used as
```solidity
(,,,, totalRa, ammCtBalance) = __calculateTotalRaAndCtBalanceWithReserve(
            self, raReserve, ctReserve, flashSwapRouter.getLvReserve(self.info.toId(), dsId)
        );
```

`flashSwapRouter.getLvReserve(self.info.toId(), dsId)` gives us the the amount of DS tokens that are reserved in the Liquidity Vault and has nothing to do with the total LP supply.

This logic is part of `redeemEarly` and `previewRedeemExpired` functionalities.
## Impact

This issue lead to unexpected result when users redeem their shares in the LV which can result in loss of funds or complete DoS of the protocol.

## Code Snippet

`__calculateTotalRaAndCtBalance()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L425
`__calculateCtBalanceWithRate()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L438
`__calculateTotalRaAndCtBalanceWithReserve()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L450
## Tool used

Manual Review

## Recommendation

`pair.totalSupply()` can be directly used to account for the LP tokens. Internal accounting is not an option here as LP can be minted by directly providing liquidity to the pair without use of Cork Protocol.

# H-04: FlashSwapRouter.emptyReserve() always returns 0 leading to excess DS in the LV to be stucked

## Summary

The Liquidity Vault holds DS tokens, which are paired with CT tokens during LP liquidation to mint RA. The vault tracks these DS tokens internally and attempts to utilize all available DS when the current DS expires. However, the current implementation of the `emptyReserve()` function is flawed, as it is expected to return the amount of DS tokens removed during the operation. Instead, it incorrectly returns the amount of DS tokens remaining, which leads to improper accounting and prevents the vault from fully utilizing the DS tokens as intended.
## Vulnerability Detail

In `VaultLib.sol` in `_liquidatedLp()` function, the following line is used to get the available DS tokens:
`uint256 reservedDs = flashSwapRouter.emptyReserve(self.info.toId(), dsId);`

In `FlashSwapRouter.sol`:
```solidity
...
using DsFlashSwaplibrary for ReserveState;
...
function emptyReserve(Id reserveId, uint256 dsId) external override onlyOwner returns (uint256 amount) {
        amount = reserves[reserveId].emptyReserve(dsId, owner());
        emit ReserveEmptied(reserveId, dsId, amount);
}
```

In `DsFlashSwap.sol`:
```solidity
function emptyReserve(ReserveState storage self, uint256 dsId, address to) internal returns (uint256 reserve) {
        reserve = emptyReservePartial(self, dsId, self.ds[dsId].reserve, to);
}
```

```solidity
function emptyReservePartial(ReserveState storage self, uint256 dsId, uint256 amount, address to)
        internal
        returns (uint256 reserve)
    {
        self.ds[dsId].ds.transfer(to, amount);

        self.ds[dsId].reserve -= amount;
        reserve = self.ds[dsId].reserve;
    }
```

`emptyReservePartial()` returns the amount left in the reserve and in case we remove all, the result will always be 0.
## Impact

Unavailability to use the excess DS token in the LV can lead to less rewards for user who want to redeem their part of the vault.  Combined with the high likelihood of this scenario happening, this should be considered as a High severity issue.

## Code Snippet

usage of `emptyReserve()` `_liquidatedLp()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L374

`FlashSwapRouter.emptyReserve()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L69

`DsFlashSwap.emptyReserve()`
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DsFlashSwap.sol#L66C5-L78C6

## Tool used

Manual Review

## Recommendation

Update `emptyReserve()` to save the amount before removing it and use it as a return value:
```diff
function emptyReserve(ReserveState storage self, uint256 dsId, address to) internal returns (uint256 reserve) {
+		reserve = self.ds[dsId].reserve;
+       emptyReservePartial(self, dsId, self.ds[dsId].reserve, to);
-       reserve = emptyReservePartial(self, dsId, self.ds[dsId].reserve, to);
}
```

# H-05: Exchange rate between RA: CT+DS is not used when liquidating LP tokens at expiry

## Summary

From README file:
```text
At expiry, the following operations will occur in the Liquidity Vault:

1. The AMM LP is redeemed to receive Cover Token + Redemption Asset
2. Any excess Depeg Swap in the Liquidity Vault is paired with Cover Token to mint Redemption Asset
3. The excess Cover Token is used to claim Redemption Asset + Pegged Asset as described above
4. End state: Only Redemption Asset + redeemed Pegged Asset remains
```
When minting RA with CT + DS, the process should follow the same exchange rate mechanism between RA and CT + DS, similar to how it's handled when depositing RA into the PSM or when redeeming RA with CT + DS through the PSM. Protocol fails to do correct conversion from CT+DS to RA when during the liquidation process described above.
## Vulnerability Detail

The `_liquidatedLp()` function in `VaultLib.sol` is called upon DS expiration. Both `redeemExpired()` and `onNewIssuance()` use this function to execute the operations outlined in the report's summary section.

```solidity
function _liquidatedLp(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter
    ) internal {
        DepegSwap storage ds = self.ds[dsId];

        // if there's no LP, then there's nothing to liquidate
        if (self.vault.config.lpBalance == 0) {
            return;
        }

        // the following things should happen here(taken directly from the whitepaper) :
        // 1. The AMM LP is redeemed to receive CT + RA
        // 2. Any excess DS in the LV is paired with CT to redeem RA
        // 3. The excess CT is used to claim RA + PA in the PSM
        // 4. End state: Only RA + redeemed PA remains

        self.vault.lpLiquidated.set(dsId);

        (uint256 raAmm, uint256 ctAmm) = __liquidateUnchecked(
            self, self.info.pair1, self.ds[dsId].ct, ammRouter, IUniswapV2Pair(ds.ammPair), self.vault.config.lpBalance
        );

        uint256 reservedDs = flashSwapRouter.emptyReserve(self.info.toId(), dsId);
        uint256 redeemAmount = reservedDs >= ctAmm ? ctAmm : reservedDs;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId); // The following line does operation 2 from above (Any excess DS in the LV is paired with CT to redeem RA)

        // if the reserved DS is more than the CT that's available from liquidating the AMM LP
        // then there's no CT we can use to effectively redeem RA + PA from the PSM
        uint256 ctAttributedToPa = reservedDs >= ctAmm ? 0 : ctAmm - reservedDs;

        uint256 psmPa;
        uint256 psmRa;

        if (ctAttributedToPa != 0) {
            (psmPa, psmRa) = PsmLibrary.lvRedeemRaPaWithCt(self, ctAttributedToPa, dsId);
        }

        psmRa += redeemAmount;

        self.vault.pool.reserve(self.vault.lv.totalIssued(), raAmm + psmRa, psmPa);
}
```

```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
        DepegSwap storage ds = self.ds[dsId];
        ds.burnBothforSelf(amount);
}
```

```solidity
function burnBothforSelf(DepegSwap storage self, uint256 amount) internal {
    Asset(self._address).burn(amount);
    Asset(self.ct).burn(amount);
}
```

The current implementation does not account for the exchange rate between RA and DS+CT when burning tokens, resulting in an incorrect amount of RA tokens being minted. Instead, it accounts RA tokens equal to the number of CT/DS tokens burned, which can cause imbalances. If the exchange rate is greater than 1, users will receive fewer RA tokens than they should, leading to reduced rewards. Conversely, if the exchange rate is less than 1, the protocol will mint too many RA tokens, providing more rewards to users than intended. This can lead to an over-distribution of RA tokens, causing imbalances in the system.
## Impact

Depending on the scenarios described in the previous section, this issue can lead to either a loss of rewards for users or an over-distribution of rewards. Over time, this imbalance could break the functionality of the protocol, as it may run out of RA tokens to properly reward all users who provided liquidity into the liquidity vault. Given the high impact on user rewards and the likelihood of occurrence, the overall severity of this issue is high.

## Code Snippet

`_liquidatedLp()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L349
`lvRedeemRaWithCtDs()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L125
`burnBothFromSelf()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DepegSwapLib.sol#L60

## Tool used

Manual Review

## Recommendation

When converting CT+DS to RA, use exchange rate to prevent such imbalances.


# H-06: Exchange rate between RA:CT+DS is not used when liquidating LP tokens during early redeem
## Summary

From README file: 
```text
Liquidity Vault tokenholders can withdraw prior to expiry, the following logic is applied to calculate how many RA they receive per LV:

1. AMM LP Liquidation   
2. Depeg Swap Pairing with Cover Token  
3. Excess Cover Token from the AMM LP liquidation is sold into the AMM to receive Redemption Asset.
4. The fraction of Redemption Asset attributed to the Liquidity Vault tokenholder is added to total amount of Redemption Asset to be withdrawn
5. The amount of Redemption Asset attributed per Liquidity Vault token from steps 1-4, minus a fee can be claimed by the Liquidity Vault tokenholder in exchange for burning their Liquidity Vault token
```
When minting RA with CT + DS, the process should follow the same exchange rate mechanism between RA and CT + DS, similar to how it's handled when depositing RA into the PSM or when redeeming RA with CT + DS through the PSM. Protocol fails to do correct conversion from CT+DS to RA when during the liquidation process described above (operation 2).
## Vulnerability Detail

The issue arises when users redeem early from the Liquidity Vault. In the `_redeemCtDsAndSellExcessCt()` function, the exchange rate between RA and CT+DS is not applied during the redemption process, leading to an incorrect amount of RA tokens being credited. This can result in over-rewarding or under-rewarding users, depending on the current exchange rate. If the exchange rate is greater than 1, users will receive fewer RA tokens than they should, leading to reduced rewards. Conversely, if the exchange rate is less than 1, the protocol will mint too many RA tokens, providing more rewards to users than intended. This can lead to an over-distribution of RA tokens, causing imbalances in the system.

```solidity
function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
        uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

        reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

         // @audit exchange rate is not used
        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
            // @audit can this 100% tolerance be exploited + one more place with 100% tolerance
            ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }
```

```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
        DepegSwap storage ds = self.ds[dsId];
        ds.burnBothforSelf(amount);
}
```

```solidity
function burnBothforSelf(DepegSwap storage self, uint256 amount) internal {
    Asset(self._address).burn(amount);
    Asset(self.ct).burn(amount);
}
```

## Impact

Depending on the scenarios described in the previous section, this issue can lead to either a loss of rewards for users or an over-distribution of rewards. Over time, this imbalance could break the functionality of the protocol, as it may run out of RA tokens to properly reward all users who provided liquidity into the liquidity vault. Given the high impact on user rewards and the likelihood of occurrence, the overall severity of this issue is high.
## Code Snippet

`_redeemCtDsAndSellExcessCt()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L318
`lvRedeemRaWithCtDs()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L125
`burnBothFromSelf()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/DepegSwapLib.sol#L60
## Tool used

Manual Review
## Recommendation

When converting CT+DS to RA, use exchange rate to prevent such imbalances.

# H-07: lockUnchecked() is used instead lockFrom() in PsmLib.repurchase() leading to wrong accounting of locked RA tokens.

## Summary

When users interact with the PSM and provide RA, the `lockFrom()` function is used to account for the deposited tokens. Conversely, when users withdraw RA from the PSM, the `unlockTo()` function is used to track the RA being withdrawn.
## Vulnerability Detail

The current implementation of the `repurchase()` function in `PsmLib.sol` fails to properly account for the RA tokens provided by the user, as it incorrectly uses `lockUnchecked()` instead of `lockFrom()`. The `lockFrom()` function not only transfers the RA tokens but also updates the locked balance, which is crucial for tracking the tokens that are available to reward users.

By using `lockUnchecked()`, the locked RA tokens are not accounted for, leading to an incorrect balance. This can result in fewer tokens being available for rewards, causing users to receive less than expected.
```solidity
function repurchase(
        State storage self,
        address buyer,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 dsId, uint256 received, uint256 feePrecentage, uint256 fee, uint256 exchangeRates) {
        DepegSwap storage ds;

        (dsId, received, feePrecentage, fee, exchangeRates, ds) = previewRepurchase(self, amount);

        // decrease PSM balance
        // we also include the fee here to separate the accumulated fee from the repurchase
        self.psm.balances.paBalance -= (received);
        self.psm.balances.dsBalance -= (received);

        // transfer user RA to the PSM/LV
        // @audit shouldnt it be lock checked, deposit -> redeemWithDs -> repurchase - locked would be 0
        self.psm.balances.ra.lockUnchecked(amount, buyer);

        // transfer user attrubuted DS + PA
        // PA
        (, address pa) = self.info.underlyingAsset();
        IERC20(pa).safeTransfer(buyer, received);

        // DS
        IERC20(ds._address).transfer(buyer, received);

        // Provide liquidity
        VaultLibrary.provideLiquidityWithFee(self, fee, flashSwapRouter, ammRouter);
    }
```

In `LvAssetLib.sol`:
```solidity
function incLocked(LvAsset storage self, uint256 amount) internal {
        self.locked = self.locked + amount;
    }

    function decLocked(LvAsset storage self, uint256 amount) internal {
        self.locked = self.locked - amount;
    }

    function lockFrom(LvAsset storage self, uint256 amount, address from) internal {
        incLocked(self, amount);
        lockUnchecked(self, amount, from);
    }

    function unlockTo(LvAsset storage self, uint256 amount, address to) internal {
        decLocked(self, amount);
        self.asErc20().transfer(to, amount);
    }

    function lockUnchecked(LvAsset storage self, uint256 amount, address from) internal {
        ERC20(self._address).transferFrom(from, address(this), amount);
    }

```
## Impact

Less rewards than expected + locked RA tokens in the contract.
## Code Snippet

`PsmLib.repurchase()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L293
## Tool used

Manual Review
## Recommendation
Use `lockFrom()` instead of `lockUnchecked()`.

# H-08: lvRedeemRaWithCtDs() does not account for the RA tokens locked in PSM

## Summary

From README:
```text
In the Vault Deposit, a portion of the Redemption Asset is used to mint Cover Token + Depeg Swap. The Redemption Asset + Cover Token is used to provide liquidity to an an Automated Market Maker (AMM). The vault receives back Liquidity Provider (LP) positions. The Depeg Swaps minted is gradually sold to the AMM via limit orders to accrue more Redemption Asset to the Vault.
```

```text
Liquidity Vault tokenholders can withdraw prior to expiry, the following logic is applied to calculate how many RA they receive per LV:

1. AMM LP Liquidation   
2. Depeg Swap Pairing with Cover Token  
3. Excess Cover Token from the AMM LP liquidation is sold into the AMM to receive Redemption Asset.
4. The fraction of Redemption Asset attributed to the Liquidity Vault tokenholder is added to total amount of Redemption Asset to be withdrawn
5. The amount of Redemption Asset attributed per Liquidity Vault token from steps 1-4, minus a fee can be claimed by the Liquidity Vault tokenholder in exchange for burning their Liquidity Vault token
```

The portion of RA used to mint CT + DS is properly accounted for in the PSM's locked RA balance. However, when users redeem early and burn CT + DS to convert them back into RA, the PSM's locked RA balance is not updated. This leads to an inconsistency in the accounting of RA, as the locked balance does not reflect the actual amount of RA available after redemption.
## Vulnerability Detail

The `deposit()` function in `VaultLib.sol` is responsible for depositing RA into the Liquidity Vault (LV). Internally, it calls `__provideLiquidityWithRatio()`, which uses `__provideLiquidity()`, and eventually calls `PsmLibrary.unsafeIssueToLv()`. This way the amount of RA tokens used to mint CT+DS is accounted in the system.

```solidity
function unsafeIssueToLv(State storage self, uint256 amount) internal {
        uint256 dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        // @audit exchange rate not used
        self.psm.balances.ra.incLocked(amount);

        ds.issue(address(this), amount);
}
```

On the other hand, during the opposite operation, such as in the `redeemEarly()` function, the RA taken when CT + DS are burned is not properly accounted for. `redeemEarly()` calls `_liquidateLpPartial()`, which uses `_redeemCtDsAndSellExcessCt()`, ultimately relying on `PsmLibrary.lvRedeemRaWithCtDs()`. However, this function burns the CT + DS tokens without adjusting the PSM's locked RA balance, failing to update the system’s balances properly.

```solidity
function lvRedeemRaWithCtDs(State storage self, uint256 amount, uint256 dsId) internal {
        DepegSwap storage ds = self.ds[dsId];
        ds.burnBothforSelf(amount);
}
```

This incorrect accounting leads to more rewards being distributed than expected for users redeeming RA + PA with CT in the PSM. Over time, this can result in the protocol running out of RA tokens, preventing future users from receiving their rightful rewards. If left unchecked, the depletion of RA tokens could severely impact the protocol's ability to fulfill its obligations, ultimately leading to liquidity issues and undermining the protocol's stability.
## Impact

Over-distribution of rewards + potential DoS
## Code Snippet

`VaultLib.deposit()`
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L191
`VaultLib.redeemEarly()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L639
## Tool used

Manual Review
## Recommendation
Use `self.psm.balances.ra.decLocked(amount);` in `lvRedeemRaWithCtDs()`


# H-09: Exchange rate between RA:CT+DS is not used providing RA tokens to liquidity vault

## Summary

In the Vault Deposit, a portion of the Redemption Asset is used to mint Cover Token + Depeg Swap. Exchange rate is not applied here and for each CT minted we lock the same amount of RA.
## Vulnerability Detail
The `deposit()` function in `VaultLib.sol` is responsible for depositing RA into the Liquidity Vault, with a portion of this RA being used to mint CT + DS for providing liquidity to the AMM pair. However, this process does not account for the actual exchange rate between RA and CT + DS, assuming a 1:1 exchange rate.

In the `__provideLiquidity()` function, the minted CT tokens and the corresponding RA are provided as liquidity to the AMM. However, when `PsmLibrary.unsafeIssueToLv()` is called to account for the locked RA used to mint CT + DS, it incorrectly assumes an exchange rate of 1:1 between RA and CT.
```solidity
function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

`PsmLibrary.unsafeIssueToLv()` is used to account for the RA token locked in order to mint the CT+DS.

```solidity
function unsafeIssueToLv(State storage self, uint256 amount) internal {
        uint256 dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        self.psm.balances.ra.incLocked(amount);

        ds.issue(address(this), amount);
    }
```

The RA amount locked is equal to the CT amount minted which is equal to exchange rate = 1. Since exchange rate can be different than one, this function should account for it and lock the correct RA based on the exchange rate.

This issue can result in users receiving either fewer or more rewards, depending on whether the exchange rate is higher or lower than 1. Apart from that users might not be incentivized to deposit RA to the vault, but instead directly min DS+CT in the PSM and provide liquidity directly to the AMM pair.
## Impact

Wrong amount of rewards distributed for users
## Code Snippet

`VaultLib.deposit()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L191
`VaultLib.__provideLiquidity()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L153
## Tool used

Manual Review

## Recommendation

Use exchange rate when depositing RA to the LV.

# H-10: Wrong accounting of locked RA when redeeming RA with DS

## Summary

Users have the option to redeem RA by providing DS + PA to the PSM. When they redeem RA, a fee is applied to the amount of RA redeemed, and this fee is used to mint CT for providing liquidity to the AMM pair. However, during this process, the protocol fails to correctly account for the amount of RA that remains locked in the PSM.

## Vulnerability Detail

In the current implementation of `Psm.redeemRaWithDs()`, two key operations are performed:

1. Sending the corresponding amount of RA to the user in exchange for DS + PA,
2. Providing liquidity with the fee generated from the RA redemption.

```solidity
function redeemRaWithDs(Id id, uint256 dsId, uint256 amount, bytes memory rawDsPermitSig, uint256 deadline)
        external
        override
        nonReentrant
        onlyInitialized(id)
        PSMWithdrawalNotPaused(id)
    {
        State storage state = states[id];
        // gas savings
        uint256 feePrecentage = psmBaseRedemptionFeePrecentage;

        (uint256 received, uint256 _exchangeRate, uint256 fee) =
            state.redeemWithDs(_msgSender(), amount, dsId, rawDsPermitSig, deadline, feePrecentage);

        VaultLibrary.provideLiquidityWithFee(state, fee, getRouterCore(), getAmmRouter());

        emit DsRedeemed(id, dsId, _msgSender(), amount, received, _exchangeRate, feePrecentage, fee);
    }
```

In`PsmLib._afterRedeemWithDs()` the amount unlocked is equal to the converted RA (from DS + PA) minus the fee.
```solidity
function _afterRedeemWithDs(
        State storage self,
        DepegSwap storage ds,
        address owner,
        uint256 amount,
        uint256 feePrecentage
    ) internal returns (uint256 received, uint256 _exchangeRate, uint256 fee) {
        IERC20(ds._address).transferFrom(owner, address(this), amount);

        _exchangeRate = ds.exchangeRate();
        received = MathHelper.calculateRedeemAmountWithExchangeRate(amount, _exchangeRate);

        // @audit-issue incorrect order of params
        fee = MathHelper.calculatePrecentageFee(received, feePrecentage);
        received -= fee;

        IERC20(self.info.peggedAsset().asErc20()).safeTransferFrom(owner, address(this), amount);

        // @audit wrong accounting of locked value
        self.psm.balances.ra.unlockTo(owner, received);
    }
```

The amount that is unlocked is the (converted RA from DS+PA) - fee.

The `VaultLib.__provideLiquidity()` function is used to send RA and CT tokens derived from the redemption fee to the AMM pair.
```solidity
function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

`PsmLibrary.unsafeIssueToLv()` accounts for the locked amount of RA from the providing liquidity. The amount is equal to the amount of CT tokens sent to the AMM.
```solidity
function unsafeIssueToLv(State storage self, uint256 amount) internal {
        uint256 dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        self.psm.balances.ra.incLocked(amount);

        ds.issue(address(this), amount);
    }
```

Consider the following scenario:
For simplicity the minted values in the example may not be accurate, but the idea is to show the wrong accounting of locked RA.

1. PSM has 1000 RA locked.
2. Alice provide 100 DS+PA and we have fee=5%
3. Fee is 5 RA, so Alice would receive 95 RA
4. After `_afterRedeemWithDs()` the locked amount is equal to 905 (1000 initial RA - 95 for Alice), fee stays locked.
5. In `__provideLiquidity()` let say 3 of those 5 RA are used to mint CT+DS.
6. `PsmLibrary.unsafeIssueToLv()` would add 3 RA to the locked amount, making the psm.balances.ra.locked = 908.

This is incorrect because the real amount of RA locked is 905 3 used to provide liquidity are already accounted from before since fee was not deducted from the initial amount
## Impact

Wrong accounting of locked RA would lead to over-distribution of rewards for users + after time last users to redeem might not be able to redeem as there wont be enough RA in the contract due to previous over-distribution. This breaks a core functionality of the protocol and the likelihood of this happening is very high, making the overall severity High.
## Code Snippet

`Psm.redeemRaWithDs()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/Psm.sol#L123
## Tool used

Manual Review

## Recommendation

Consider not accounting the RA used to mint CT+DS in this scenario as it is already accounted. Directly changing `unsafeIssueToLv()` may not be an option as it is used in other places as well. Another option is to remove the fee from the locked amount in `_afterRedeemWithDs()`.

# H-11: Wrong accounting of locked RA when repurchasing DS+PA with RA
## Summary

Users have the option to repurchase DS + PA by providing RA to the PSM. A portion of the RA provided is taken as a fee, and this fee is used to mint CT + DS for providing liquidity to the AMM pair.
## Vulnerability Detail

NOTE: Currently `PsmLib.sol` incorrectly uses `lockUnchecked()` for the amount of RA provided by the user. As discussed with sponsor it should be `lockFrom()` in order to account for the RA provided.

After initially locking the RA provided, part of this amount is used to provide liquidity to the AMM via `VaultLibrary.provideLiquidityWithFee()`

```solidity
function repurchase(
        State storage self,
        address buyer,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 dsId, uint256 received, uint256 feePrecentage, uint256 fee, uint256 exchangeRates) {
        DepegSwap storage ds;

        (dsId, received, feePrecentage, fee, exchangeRates, ds) = previewRepurchase(self, amount);

        // decrease PSM balance
        // we also include the fee here to separate the accumulated fee from the repurchase
        self.psm.balances.paBalance -= (received);
        self.psm.balances.dsBalance -= (received);

        // transfer user RA to the PSM/LV
        // @audit-issue shouldnt it be lock checked, deposit -> redeemWithDs -> repurchase - locked would be 0
        self.psm.balances.ra.lockUnchecked(amount, buyer);

        // transfer user attrubuted DS + PA
        // PA
        (, address pa) = self.info.underlyingAsset();
        IERC20(pa).safeTransfer(buyer, received);

        // DS
        IERC20(ds._address).transfer(buyer, received);

        // Provide liquidity
        VaultLibrary.provideLiquidityWithFee(self, fee, flashSwapRouter, ammRouter);
    }
```

`provideLiqudityWithFee` internally uses `__provideLiquidityWithRatio()` which calculates the amount of RA that should be used to mint CT+DS in order to be able to provide liquidity.
```solidity
function __provideLiquidityWithRatio(
        State storage self,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 ra, uint256 ct) {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ctRatio);

        __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);
    }
```

`__provideLiquidity()` uses `PsmLibrary.unsafeIssueToLv()` to account the RA locked.
```solidity
function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

RA locked is incremented with the amount of CT tokens minted.
```solidity
function unsafeIssueToLv(State storage self, uint256 amount) internal {
        uint256 dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        self.psm.balances.ra.incLocked(amount);

        ds.issue(address(this), amount);
    }

```

Consider the following scenario:
For simplicity the minted values in the example may not be accurate, but the idea is to show the wrong accounting of locked RA.

1. PSM has 1000 RA locked.
2. Alice repurchase 100 DS+PA with providing 100 RA and we have fee=5% making the fee = 5
3. PSM will have 1100 RA locked and ra.locked would be 1100 also.
5. In `__provideLiquidity()` let say 3 of those 5 RA are used to mint CT+DS.
6. `PsmLibrary.unsafeIssueToLv()` would add 3 RA to the locked amount, making the psm.balances.ra.locked = 1103 while the real balance would still be 1100.

## Impact

Wrong accounting of locked RA would lead to over-distribution of rewards for users + after time last users to redeem might not be able to redeem as there wont be enough RA in the contract due to previous over-distribution. This breaks a core functionality of the protocol and the likelihood of this happening is very high, making the overall severity High.
## Code Snippet

`PsmLib.repurchase()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L293
## Tool used

Manual Review
## Recommendation

Consider either not accounting for the fee used to mint CT+DS as it is already accounted or do not initially account for the fee when users provide RA.

# H-12: Initial AMM price ratio can be manipulated

## Summary

When providing liquidity to the AMM during new DS issuance, when depositing to the vault, or with fees, the protocol checks the current price ratio of RA:CT in order to calculate how much RA and CT are needed to be sent to the pair. Current implementation account for the case where the pool might be empty and calculating the ratio based on the reserves in it wont work. This provides an opportunity for attackers to manipulate the price ratio before the first provision of liquidity.

## Vulnerability Detail

`____getAmmCtPriceRatio()` is used to get the current RA:CT price ration in the AMM.

```solidity
function __getAmmCtPriceRatio(State storage self, IDsFlashSwapCore flashSwapRouter, uint256 dsId)
        internal
        view
        returns (uint256 ratio)
    {
        // This basically means that if the reserve is empty, then we use the default ratio supplied at deployment
        ratio = self.ds[dsId].exchangeRate() - self.vault.initialDsPrice;

        // will always fail for the first deposit
        try flashSwapRouter.getCurrentPriceRatio(self.info.toId(), dsId) returns (uint256, uint256 _ctRatio) {
            ratio = _ctRatio;
        } catch {}
    }
```

Since Ethereum’s mainnet has a public mempool, attackers can monitor when the first vault deposit is about to occur. It is expected that the initial liquidity provision will use a specific price ratio set by the protocol. However,since the AMM pair is empty, an attacker can frontrun the transaction by providing a small amount of liquidity to the pool first. This allows the attacker to set an incorrect RA:CT price ratio that deviates significantly from the expected initial ratio.

Consider the following scenario: (for simplicity `AMM X*Y =K` is not correct in the current example, but is close to a correct one)
	1. New issuance has occured with exchange rate between RA:CT+DS = 1.
	2. Attacker mints 10 CT + 10 DS by providing 10 RA
	3. Another user decide to deposit 50 RA to the LV
	4. It is expected that liquidty sent to the AMM would be ~50% RA ~ 50% CT
	5. Attacker frontrun the deposit transaction and deposits 1 RA and 0.1 CT
	6. Now the price ration between RA:CT would be much higher than expected.
	7. Only small amount of the user's 50 RA would be used to mint CT and almost all of it(RA) are going to be provided to the pair
	8. New pair reserves are going to be ~45 RA and 4.5 CT
	9. Now attacker can provide his CT tokens in order to get more RA than initially invested.

This deviation from the initial price would persist over time, undermining the protocol’s core functionality and making it unusable. The issue arises because the protocol does not provide initial liquidity to the newly created pair, leaving it vulnerable to manipulation. Without setting the initial price through a controlled liquidity provision, attackers can easily deviate the RA:CT price ratio, causing long-term imbalances that disrupt the protocol's intended operations.
## Impact

Protocol is going to be completely unusable as intended.

## Code Snippet

`VaultLib.onNewIssuance()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L92
`VaultLib.deposit()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L191
`VaultLib.provideLiquidityWithFee()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L602
`VaultLib.__getAmmCtPriceRatio()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L139
## Tool used

Manual Review

## Recommendation

Always provide initial liquidity to newly created pairs to set the initial price ratio.



# H-13: On-chain calculated tolerance does not provide slippage protection when depositing into LV
## Summary

When users deposit into the Liquidity Vault (LV), a portion of their RA is converted to CT + DS tokens, and CT + RA is provided as liquidity to the AMM pair. The protocol sets a tolerance level to ensure that at least a certain amount of RA and CT is successfully provided. However, this tolerance is calculated on-chain using the current state of the AMM reserves, which are subject to change. An attacker could frontrun the transaction, manipulating the reserves and leave the user deposit RA in a state where RA price ratio to CT is very low.
## Vulnerability Detail 
The function `__addLiquidityToAmmUnchecked()` is responsible for adding liquidity to the AMM by calculating the tolerance for the amount of RA and CT tokens to be provided. The tolerance is determined on-chain using the current state of the AMM's reserves:
```solidity
    function __addLiquidityToAmmUnchecked(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        (uint256 raTolerance, uint256 ctTolerance) =
            MathHelper.calculateWithTolerance(raAmount, ctAmount, MathHelper.UNIV2_STATIC_TOLERANCE);

        ERC20(raAddress).approve(address(ammRouter), raAmount);
        ERC20(ctAddress).approve(address(ammRouter), ctAmount);

        (address token0, address token1, uint256 token0Amount, uint256 token1Amount) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raAmount, ctAmount);
        (,, uint256 token0Tolerance, uint256 token1Tolerance) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raTolerance, ctTolerance);

        (,, uint256 lp) = ammRouter.addLiquidity(
            token0, token1, token0Amount, token1Amount, token0Tolerance, token1Tolerance, address(this), block.timestamp
        );

        self.vault.config.lpBalance += lp;
    }
```

The `raAmount` and `ctAmount` to be provided are calculated based on the current reserves of the pair through the `__provideLiquidityWithRatio()` function:
```solidity
 function __provideLiquidityWithRatio(
        State storage self,
        uint256 amount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal returns (uint256 ra, uint256 ct) {
        uint256 dsId = self.globalAssetIdx;

        uint256 ctRatio = __getAmmCtPriceRatio(self, flashSwapRouter, dsId);

        (ra, ct) = MathHelper.calculateProvideLiquidityAmountBasedOnCtPrice(amount, ctRatio);

        __provideLiquidity(self, ra, ct, flashSwapRouter, ctAddress, ammRouter, dsId);
    }
```

```solidity
function __provideLiquidity(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        IDsFlashSwapCore flashSwapRouter,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        uint256 dsId
    ) internal {
        // no need to provide liquidity if the amount is 0
        if (raAmount == 0 && ctAmount == 0) {
            return;
        }

        PsmLibrary.unsafeIssueToLv(self, ctAmount);

        __addLiquidityToAmmUnchecked(self, raAmount, ctAmount, self.info.redemptionAsset(), ctAddress, ammRouter);

        _addFlashSwapReserve(self, flashSwapRouter, self.ds[dsId], ctAmount);
    }
```

The vulnerability arises because the RA and CT amounts (along with the tolerance levels) are calculated on-chain based on the current state of the AMM reserves. This opens up the possibility for an attacker to frontrun the transaction and manipulate the reserves of the AMM pair just before the user’s deposit is processed. By doing so, the attacker can create a sandwich attack by manipulating the price ratio.

Consider the following scenario:

1. Current price ratio is ~1:1 and there is not much liquidity in the pair
2. User deposits large amount of RA:CT tokens
3. Attacker frontruns the transaction and execute swap to change the ratio between assets to make RA being cheap compared to CT.
4. User transaction gets executed, providing large amount of RA and low amount of CT (because of the new price ratio)
5. Attacker executes his second transaction, swapping his CT to get RA.
## Impact

This would results in imbalances in the vault regarding the CT and RA balances which later can lead to loss of rewards for users as they would have provided much more RA than they would get when liquidating LP. Likelihood of this happening is high as attacker has direct financial incentives, making the overall severity High.

## Code Snippet

`VaultLib.deposit()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L191
`VaultLib.__addLiquidityToAmmUnchecked()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L55
`VaultLib.__provideLiquidityWithRatio()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L123
`VaultLib.__provideLiquidity()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L153

## Tool used

Manual Review

## Recommendation

Provide off-chain tolerance calculation which is provided when user deposit into LV.

# H-14: Buying DS with RA does not account for the exhange rate between RA: CT+DS
## Summary

From README file:

```text
Buying Depeg Swaps:

1. Buyer sends Redemption Asset into the swap contract 
2. A contract withdraws more Redemption Asset from the AMM
3. The Redemption Asset is used to mint Cover Token and Depeg Swap
4. Depeg Swap is sent to the buyer
5. The Cover Token is sold for Redemption Asset to return the amount from step 2
```

From 3 - RA is used to mint CT+DS but the protocol does not take into consideration the exchange rate when minting CT+DS, leading to a locked funds.
## Vulnerability Detail

The function `__afterFlashswapBuy()` is used during a flash swap to mint CT + DS, send the DS tokens to the user, and return CT tokens to the AMM in exchange for the borrowed RA. However, a significant issue arises due to the unchecked return value from `psm.depositPsm()`.

In this function, `dsAttributed` is assumed to represent the amount of DS the user should receive based on the amount of RA they provided in the `swapRaforDs()` operation. The assumption here is that the exchange rate between RA and DS is 1:1, which is incorrect.
```solidity
function __afterFlashswapBuy(
        ReserveState storage self,
        Id reserveId,
        uint256 dsId,
        address caller,
        uint256 dsAttributed
    ) internal {
        AssetPair storage assetPair = self.ds[dsId];
        assetPair.ra.approve(owner(), dsAttributed);

        IPSMcore psm = IPSMcore(owner());
        psm.depositPsm(reserveId, dsAttributed);

        // should be the same, we don't compare with the RA amount since we maybe dealing
        // with a non-rebasing token, in which case the amount deposited and the amount received will always be different
        // so we simply enforce that the amount received is equal to the amount attributed to the user

        // send caller their DS
        assetPair.ds.transfer(caller, dsAttributed);
        // repay flash loan
        assetPair.ct.transfer(msg.sender, dsAttributed);
    }
```

The `Psm.depositPsm()` function internally calls `PsmLib.deposit()` to calculate the correct amount of DS + CT tokens to mint based on the current exchange rate between RA and DS. However, `__afterFlashswapBuy()` does not account for the actual exchange rate and assumes that providing `X` amount of RA will mint `X` amount of DS, leading to a mismatch.
```solidity
function deposit(State storage self, address depositor, uint256 amount)
        internal
        returns (uint256 dsId, uint256 received, uint256 _exchangeRate)
    {
        if (amount == 0) {
            revert ICommon.ZeroDeposit();
        }

        dsId = self.globalAssetIdx;

        DepegSwap storage ds = self.ds[dsId];

        Guard.safeBeforeExpired(ds);
        _exchangeRate = ds.exchangeRate();

        received = MathHelper.calculateDepositAmountWithExchangeRate(amount, _exchangeRate);

        self.psm.balances.ra.lockFrom(amount, depositor);

        ds.issue(depositor, received);
    }
```

In `__afterFlashswapBuy()`, the actual amount of DS + CT tokens minted is not considered, as it assumes a 1:1 exchange rate (i.e., 1 RA = 1 DS). This assumption can lead to two issues:

1. Incorrect Token Distribution: If the exchange rate is not 1:1, the amount of DS minted will differ from the `dsAttributed` amount. This discrepancy leads to a miscalculation in the number of tokens sent to the user and those repaid in the flash loan.
2. Locked CT + DS Tokens: Since the function does not correctly account for the exchange rate, CT + DS tokens could remain locked in the contract, causing inefficiencies and potentially leading to incorrect balances being held in the swap contract.
## Impact

An incorrect amount of tokens is distributed, or CT + DS tokens become locked in the contract whenever the exchange rate differs from 1. Since the current implementation assumes a 1:1 exchange rate, this issue will occur anytime the actual exchange rate is different, making the likelihood of this happening **certain** whenever the exchange rate isn't exactly 1:1.
## Code Snippet
`FlashSwapRouter.__afterFlashswapBuy()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/flash-swaps/FlashSwapRouter.sol#L357
`PsmLib.deposit()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/PsmLib.sol#L87
## Tool used

Manual Review

## Recommendation

A potential solution with minimal changes would be to check the return values of `psm.deposit()` and handle any unused CT + DS tokens appropriately, such as converting them back to RA. This ensures that no tokens remain locked unnecessarily.
Alternatively, a more comprehensive approach would involve calculating the amount of DS tokens that can be minted based on the current exchange rate upfront. This would allow the existing logic in `__afterFlashswapBuy()` to remain unchanged while ensuring accurate token distribution.


# H-16: Missing slippage protection when redeeming rewards from LV
## Summary

Slippage protection is crucial for ensuring users receive the rewards they expect when removing liquidity or executing a swap in an AMM pair. However, the current protocol implementation does not offer any slippage protection mechanism. Instead, it enforces a 100% tolerance by design to prevent transaction failures, which leaves users vulnerable to receiving less than expected due to unfavorable price movements or liquidity changes.

To enhance user security, the protocol should allow users to specify their acceptable slippage tolerance during transactions. This way, users can control how much price fluctuation they are willing to accept, ensuring they can opt out of transactions if the slippage exceeds their chosen limit.
## Vulnerability Detail

Both the `_redeemCtDsAndSellExcessCt()` and `__liquidateUnchecked()` functions execute liquidity removal and swap operations on the AMM pair without providing slippage protection. This lack of slippage safeguards exposes the protocol to **sandwich attacks**, where an attacker can manipulate the price between the time a transaction is initiated and when it is executed.

```solidity
function _redeemCtDsAndSellExcessCt(
        State storage self,
        uint256 dsId,
        IUniswapV2Router02 ammRouter,
        IDsFlashSwapCore flashSwapRouter,
        uint256 ammCtBalance
    ) internal returns (uint256 ra) {
        uint256 reservedDs = flashSwapRouter.getLvReserve(self.info.toId(), dsId);

        uint256 redeemAmount = reservedDs >= ammCtBalance ? ammCtBalance : reservedDs;

        reservedDs = flashSwapRouter.emptyReservePartial(self.info.toId(), dsId, redeemAmount);

        ra += redeemAmount;
        PsmLibrary.lvRedeemRaWithCtDs(self, redeemAmount, dsId);

        uint256 ctSellAmount = reservedDs >= ammCtBalance ? 0 : ammCtBalance - reservedDs;

        DepegSwap storage ds = self.ds[dsId];
        address[] memory path = new address[](2);
        path[0] = ds.ct;
        path[1] = self.info.pair1;

        ERC20(ds.ct).approve(address(ammRouter), ctSellAmount);

        if (ctSellAmount != 0) {
            // 100% tolerance, to ensure this not fail
            ra += ammRouter.swapExactTokensForTokens(ctSellAmount, 0, path, address(this), block.timestamp)[1];
        }
    }

```

```solidity
function __liquidateUnchecked(
        State storage self,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter,
        IUniswapV2Pair ammPair,
        uint256 lp
    ) internal returns (uint256 raReceived, uint256 ctReceived) {
        ammPair.approve(address(ammRouter), lp);

        // amountAMin & amountBMin = 0 for 100% tolerence
        (raReceived, ctReceived) =
            ammRouter.removeLiquidity(raAddress, ctAddress, lp, 0, 0, address(this), block.timestamp);

        (raReceived, ctReceived) = MinimalUniswapV2Library.reverseSortWithAmount224(
            ammPair.token0(), ammPair.token1(), raAddress, ctAddress, raReceived, ctReceived
        );

        self.vault.config.lpBalance -= lp;
    }
```
## Impact

Loss of funds for the users who withdraw their rewards.
## Code Snippet
`VaultLib._redeemCtDsAndSellExcessCt()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L318
`VaultLib.__liquidateUnchecked()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L270
## Tool used

Manual Review
## Recommendation

Slippage parameters should be included in the transaction and specified by the users to prevent such scenarios.
# M-01: Possibility of DoS when issuing new DS

## Summary

Smart contracts can be created both by other contracts and by regular accounts. In both cases, the address for the new contract is computed the same way: as a function of the sender’s own address and a nonce. Every account has an associated nonce: for regular accounts it is increased on every transaction, while for contract accounts it is increased on every contract creation. Nonces cannot be reused, and they must be sequential. This means it is possible to predict the address where the _next_ created contract will be deployed

When new DS is issued via `ModuleCore.issueNewDs()`, the DS and CT token contracts are deployed using the standard `create` opcode. The address of these new contracts are deterministically generated based on `new_address = hash(sender, nonce)`. The CT contract address will be used to create a new AMM pair. If an attempt is made to create a pair for RA and CT that already exists, the transaction will revert, as a pair for these specific token addresses cannot be duplicated.
## Vulnerability Detail

Since Ethereum mainnet has a public mempool, an attacker can monitor when the protocol admin is about to issue a new DS. By calculating the addresses of the newly created CT and DS tokens (which are deterministic based on the `sender` and `nonce`), the attacker can front-run the transaction and create the AMM pair for these tokens before the protocol does. This would cause the line `address ammPair = getAmmFactory().createPair(ra, ct);` to revert, as the pair already exists.

To counter this, the protocol would need to increment its nonce by calling `ModuleCore.initialize()`, which deploys a new Liquidity Vault to change the nonce. However, the attacker can repeat this process indefinitely, creating the pair before the protocol does each time. Since the cost of issuing a new vault and creating a DS is higher than the attacker’s gas cost to create a new pair, the attacker could exploit this repeatedly.

Another scenario involves the attacker preemptively deploying a large number of AMM pairs with potential future DS and CT addresses (based on predicted nonces). This would force the protocol to spend even more gas incrementing the nonce multiple times by initializing new vaults until it reaches a nonce that hasn't been used by the attacker. This significantly increases the gas costs and complexity for the protocol to issue new DS tokens.

`issueNewDs()` uses to AssetFactory's `deploysSwapAssets()`
```solidity
function issueNewDs(Id id, uint256 expiry, uint256 exchangeRates, uint256 repurchaseFeePrecentage)
        external
        override
        onlyConfig
        onlyInitialized(id)
    {
        if (repurchaseFeePrecentage > 5 ether) {
            revert InvalidFees();
        }
        State storage state = states[id];

        address ra = state.info.pair1;

        uint256 prevIdx = state.globalAssetIdx++;
        uint256 idx = state.globalAssetIdx;

        (address ct, address ds) = IAssetFactory(SWAP_ASSET_FACTORY).deploySwapAssets(
            ra, state.info.pair0, address(this), expiry, exchangeRates
        );

		// This would fail if there is already created pair for the RA and CT addresses
        address ammPair = getAmmFactory().createPair(ra, ct);

        PsmLibrary.onNewIssuance(state, ct, ds, ammPair, idx, prevIdx, repurchaseFeePrecentage);

        getRouterCore().onNewIssuance(id, idx, ds, ammPair, 0, ra, ct);

        VaultLibrary.onNewIssuance(state, prevIdx, getRouterCore(), getAmmRouter());

        emit Issued(id, idx, expiry, ds, ct, ammPair);
    }


```

```solidity
function deploySwapAssets(address ra, address pa, address owner, uint256 expiry, uint256 psmExchangeRate)
        external
        override
        onlyOwner
        notDelegated
        returns (address ct, address ds)
    {
        Pair memory asset = Pair(pa, ra);

        // prevent deploying a swap asset of a non existent pair, logically won't ever happen
        // just to be safe
        if (lvs[asset.toId()] == address(0)) {
            revert NotExist(ra, pa);
        }

        string memory pairname = string(abi.encodePacked(Asset(ra).name(), "-", Asset(pa).name()));

        ct = address(new Asset(CT_PREFIX, pairname, owner, expiry, psmExchangeRate));
        ds = address(new Asset(DS_PREFIX, pairname, owner, expiry, psmExchangeRate));

        swapAssets[Pair(pa, ra).toId()].push(Pair(ct, ds));

        deployed[ct] = true;
        deployed[ds] = true;

        emit AssetDeployed(ra, ct, ds);
    }
```
## Impact

This can lead to complete DoS for the system and lock of funds for users who have decided to not redeem their rewards in the expired DS period. As there is no direct incentive for the attacker, the likelihood of this happening is Low, but the impact would be high because of the consequences described in the previous sentence, making the overall severity medium.
## Code Snippet

`ModuleCore.initialNewDS()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/ModuleCore.sol#L57
`AssetFactory.deploySwapAssets()`
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/assets/AssetFactory.sol#L140
`AssetFactory.deployLv()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/core/assets/AssetFactory.sol#L175
## Tool used

Manual Review
## Recommendation

Consider this scenario that pair may have already been created and use `try/catch` when trying to deploy new pair.


# M-02: Providing liquidity to the AMM does not check the return value of actually provided tokens leading to locked funds.

## Summary

When providing liquidity to an AMM pair, the protocol specifies both the desired amount of tokens to be provided and a minimum amount to be accepted. Any difference between the two—meaning the amount not used by the AMM—should be properly accounted for within the protocol, as it is not taken by the AMM.

## Vulnerability Detail

The `__addLiquidityToAmmUnchecked` function is used to provide liquidity to the RA:CT AMM pair. In the current implementation, `raTolerance` and `ctTolerance` are calculated based on the reserves of the pair during the current transaction with 1% slippage tolerance. The amounts to be provided are determined by the current price ratio in the pair, which ensures that the amounts are almost always exactly what the AMM expects to maintain the `X * Y = K` constant product formula.
```solidity
function __addLiquidityToAmmUnchecked(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        (uint256 raTolerance, uint256 ctTolerance) =
            MathHelper.calculateWithTolerance(raAmount, ctAmount, MathHelper.UNIV2_STATIC_TOLERANCE);

        ERC20(raAddress).approve(address(ammRouter), raAmount);
        ERC20(ctAddress).approve(address(ammRouter), ctAmount);

        (address token0, address token1, uint256 token0Amount, uint256 token1Amount) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raAmount, ctAmount);
        (,, uint256 token0Tolerance, uint256 token1Tolerance) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raTolerance, ctTolerance);

        (,, uint256 lp) = ammRouter.addLiquidity(
            token0, token1, token0Amount, token1Amount, token0Tolerance, token1Tolerance, address(this), block.timestamp
        );

        self.vault.config.lpBalance += lp;
    }

```

The current implementation does not check the actual amounts used by the AMM when providing liquidity. As a result, small differences (1-2 wei of the corresponding token) between the provided amount and the actual amount used by the AMM may remain locked in the contract. These differences arise from rounding in the RA:CT 
price ratio calculations and the corresponding amounts that should be provided. Over time, these small discrepancies could accumulate, leading to higher amount of locked tokens in the contract.

PoC:

Adjust the `__addLiquidityToAmmUnchecked()` function to:

```solidity 
function __addLiquidityToAmmUnchecked(
        State storage self,
        uint256 raAmount,
        uint256 ctAmount,
        address raAddress,
        address ctAddress,
        IUniswapV2Router02 ammRouter
    ) internal {
        (uint256 raTolerance, uint256 ctTolerance) =
            MathHelper.calculateWithTolerance(raAmount, ctAmount, MathHelper.UNIV2_STATIC_TOLERANCE);

        ERC20(raAddress).approve(address(ammRouter), raAmount);
        ERC20(ctAddress).approve(address(ammRouter), ctAmount);

        (address token0, address token1, uint256 token0Amount, uint256 token1Amount) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raAmount, ctAmount);
        (,, uint256 token0Tolerance, uint256 token1Tolerance) =
            MinimalUniswapV2Library.sortTokensUnsafeWithAmount(raAddress, ctAddress, raTolerance, ctTolerance);
        
        uint256 lp;
        // add one more block to avoid stack too deep errors
        {
            uint256 actual0;
            uint256 actual1;
            (actual0, actual1, lp) = ammRouter.addLiquidity(
                token0, token1, token0Amount, token1Amount, token0Tolerance, token1Tolerance, address(this), block.timestamp
            );
            if(actual0 != token0Amount || actual1 != token1Amount){
                revert();
            }
        }
        self.vault.config.lpBalance += lp;
    }
```

Running the tests with the following function would result in some tests failing due to this difference in provided and used amounts.
## Impact

The impact of these small amounts of locked funds is not significant on their own, but due to the compound effect over time and the high likelihood of this happening with each liquidity provision, the overall severity of the issue should be considered Medium.
## Code Snippet

`VaultLib.__addLiquidityToAmmUnchecked()`:
https://github.com/sherlock-audit/2024-08-cork-protocol/blob/db23bf67e45781b00ee6de5f6f23e621af16bd7e/Depeg-swap/contracts/libraries/VaultLib.sol#L55
## Tool used

Manual Review
## Recommendation

To handle the small differences between the provided and actual amounts used by the AMM, the return values of the `addLiquidity()` function should be checked, as shown in the adjusted `__addLiquidityToAmmUnchecked()` function. This allows the protocol to detect any discrepancies and take appropriate action.

Depending on the protocol's decision, these leftover funds can either be:
*  Returned to users to prevent token loss, ensuring they are not penalized by the rounding differences.
* Accounted for by the protocol and later used for liquidity provision or distributed as rewards, thereby ensuring the funds are not wasted and remain within the protocol’s ecosystem.

