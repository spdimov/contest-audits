
# H01: ASK Orders are not able to be settled due to wrong checks

## Summary

When a user wants to purchase points in a specific marketplace at a price they are willing to pay, they can create a **BID Offer**. Conversely, users who are ready to sell their points at that price can place an **ASK Order**. Once the Token Generation Event (TGE) occurs, it is the responsibility of the sellers to fulfill the deal and transfer the points to the buyer. Sellers are motivated to complete the transaction because they have posted collateral when they created the ASK Order, ensuring their commitment to the deal.
In code terms, Takers need to transfer the points token via `settleAskTaker()` function in `DeliveryPlace.sol`. Currently takers are not able to call this function due to wrong permission checks.

## Vulnerability Details

As correctly stated in the function's NatSpec, the caller must be the stock authority because the TAKER is responsible for settling the transaction. However, the check on line `#361` is incorrect: `_msgSender() != offerInfo.authority`. In this case, `offerInfo.authority` refers to the MAKER, who is waiting for the TAKER to deliver the points. The logic should ensure that the correct entity (the TAKER) is being validated for the settlement process.

```solidity
	/**
     * @notice Settle ask taker
     * @dev caller must be stock authority
     * @dev market place status must be AskSettling
     * @param _stock stock address
     * @param _settledPoints settled points
     * @notice _settledPoints must be less than or equal to stock points
     */
 function settleAskTaker(address _stock, uint256 _settledPoints) external {
        IPerMarkets perMarkets = tadleFactory.getPerMarkets();
        StockInfo memory stockInfo = perMarkets.getStockInfo(_stock);

        (
            OfferInfo memory offerInfo,
            MakerInfo memory makerInfo,
            MarketPlaceInfo memory marketPlaceInfo,
            MarketPlaceStatus status
        ) = getOfferInfo(stockInfo.preOffer);

        if (stockInfo.stockStatus != StockStatus.Initialized) {
            revert InvalidStockStatus();
        }

        if (marketPlaceInfo.fixedratio) {
            revert FixedRatioUnsupported();
        }
        if (stockInfo.stockType == StockType.Bid) {
            revert InvalidStockType();
        }
        if (_settledPoints > stockInfo.points) {
            revert InvalidPoints();
        }

        if (status == MarketPlaceStatus.AskSettling) {
            if (_msgSender() != offerInfo.authority) {
                revert Errors.Unauthorized();
            }
        } else {
            if (_msgSender() != owner()) {
                revert Errors.Unauthorized();
            }
            if (_settledPoints > 0) {
                revert InvalidPoints();
            }
        }

        uint256 settledPointTokenAmount = marketPlaceInfo.tokenPerPoint *
            _settledPoints;
        ITokenManager tokenManager = tadleFactory.getTokenManager();
        if (settledPointTokenAmount > 0) {
            tokenManager.tillIn(
                _msgSender(),
                marketPlaceInfo.tokenAddress,
                settledPointTokenAmount,
                true
            );

            tokenManager.addTokenBalance(
                TokenBalanceType.PointToken,
                offerInfo.authority,
                makerInfo.tokenAddress,
                settledPointTokenAmount
            );
        }

        uint256 collateralFee = OfferLibraries.getDepositAmount(
            offerInfo.offerType,
            offerInfo.collateralRate,
            stockInfo.amount,
            false,
            Math.Rounding.Floor
        );

        if (_settledPoints == stockInfo.points) {
            tokenManager.addTokenBalance(
                TokenBalanceType.RemainingCash,
                _msgSender(),
                makerInfo.tokenAddress,
                collateralFee
            );
        } else {
            tokenManager.addTokenBalance(
                TokenBalanceType.MakerRefund,
                offerInfo.authority,
                makerInfo.tokenAddress,
                collateralFee
            );
        }

        perMarkets.settleAskTaker(
            stockInfo.preOffer,
            _stock,
            _settledPoints,
            settledPointTokenAmount
        );

        emit SettleAskTaker(
            makerInfo.marketPlace,
            offerInfo.maker,
            _stock,
            stockInfo.preOffer,
            _msgSender(),
            _settledPoints,
            settledPointTokenAmount,
            collateralFee
        );
    }
```

Using the protocol test file, I have added another test case which shows the described issue.
NOTE: need to add the following line: `import {MarketPlaceStatus} from "../src/interfaces/ISystemConfig.sol";`

```solidity
function testSettleAskTakerPoC() public {
        vm.startPrank(user);
       
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Bid,
                OfferSettleType.Turbo
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        systemConfig.updateMarketPlaceStatus("Backpack", MarketPlaceStatus.AskSettling);
       
        vm.startPrank(user2);
        mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
        // Next call would revert with Errors.Unauthorized()
        deliveryPlace.settleAskTaker(stock1Addr, 500);
    }
```
## Impact

This issue prevents any ASK orders from being settled. As a result, Takers will be unable to complete their transactions, leading to the loss of their collateral. At the same time, Makers will not receive the points tokens they are expecting, which means they will also miss out on the newly airdropped tokens tied to those points. This creates significant financial losses for both Takers and Makers, severely disrupting the functionality of the marketplace.

## Tools Used

Manual review, Foundry.

## Recommendations

Instead of `_msgSender() != offerInfo.authority`, the correct check would be `_msgSender() != stockInfo.authority`.



# H02: Wrong accounting of points token for BID Makers.

## Summary
When a user wants to purchase points in a specific marketplace at a price they are willing to pay, they can create a **BID Offer**. Conversely, users who are ready to sell their points at that price can place an **ASK Order**. Once the Token Generation Event (TGE) occurs, sellers are responsible for fulfilling the deal by transferring the points to the buyer. Sellers are incentivized to complete the transaction because they have posted collateral when they created the ASK Order, ensuring their commitment.

In code terms, Takers must transfer the points tokens via the `settleAskTaker()` function in `DeliveryPlace.sol`. The transfer process isn't direct—Takers transfer the tokens to the protocol, and the virtual balance of the Maker is updated in `TokenManager.sol`. However, there is an issue: the `settleAskTaker()` function currently does not account correctly when Takers settle, leading to potential discrepancies in the Maker's balance and affecting the overall transaction integrity.

## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L384
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L109

On line `#384` in `DeliveryPlace.sol` the wrong token address is updated. As we are updating the balance for the PointToken we must provide the corresponding address.

```solidity
	tokenManager.addTokenBalance(
        TokenBalanceType.PointToken,
        offerInfo.authority,
        makerInfo.tokenAddress,
        settledPointTokenAmount
    );
```

`makerInfo.tokenAddress` is the address of the collateral token which is set initially when Offer is created by a Maker in `createOffer()` in `PreMarkets.sol` on line `#109`.

Using the protocol test file, I have added another test case which shows the described issue.
NOTE: need to add the following line: `import {MarketPlaceStatus} from "../src/interfaces/ISystemConfig.sol";`

```solidity
function testSettleAskTakerPoC() public {
        vm.startPrank(user);
       
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Bid,
                OfferSettleType.Turbo
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        systemConfig.updateMarketPlaceStatus("Backpack", MarketPlaceStatus.AskSettling);
       
        vm.startPrank(user2);
        mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
        // Next call would revert with Errors.Unauthorized()
        deliveryPlace.settleAskTaker(stock1Addr, 500);
        vm.stopPrank();
        
        // log accounted value = 0
        console.log(tokenManager.userTokenBalanceMap(user, address(mockPointToken), TokenBalanceType.PointToken));
        
        vm.prank(user);
        tokenManager.withdraw(address(mockPointToken), TokenBalanceType.PointToken);
        
        // Fails as user balance stays the same is initial
        assertGt(mockPointToken.balanceOf(user), 100000000 * 10 ** 18);
    }
```
## Impact

The incorrect accounting in the `settleAskTaker()` function poses a significant risk to the protocol. Users could potentially exploit this vulnerability to withdraw collateral tokens instead of the intended point tokens. This could lead to severe financial losses for the protocol, as collateral meant to secure transactions could be drained improperly. Moreover, the integrity of the marketplace would be compromised.
## Tools Used

Manual review, Foundry.

## Recommendations

Instead of `makerInfo.tokenAddress` which is the address of the collateral token, user `marketPlaceInfo.tokenAddress` which is the address of the point token.


# H03: Inconsistent approval and withdraw logic can lead to user funds stuck.

## Summary

Due to an inconsistency in the token approval process for different token types (e.g., WETH vs. other ERC20 tokens) and a mismatch between the NatSpec documentation and the actual code implementation, there is a risk that user funds could become stuck.

## Vulnerability Details
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/CapitalPool.sol#L24
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/TokenManager.sol#L137

The `approve` function in the provided code is intended to be called exclusively by the token manager contract. This is evident from the NatSpec comment (`@notice only can be called by token manager`). The function is designed to approve an unlimited allowance (`type(uint256).max`) of a specified token for the token manager.

However, the function lacks an Access Control List (ACL) or any other access control mechanism to enforce this restriction. Without such controls, any external contract or user could potentially call this function, leading to unintended or unauthorized approvals.

```solidity
/**
 * @dev Approve token for token manager
 * @notice only can be called by token manager
 * @param tokenAddr address of token
 */
function approve(address tokenAddr) external {
        address tokenManager = tadleFactory.relatedContracts(
            RelatedContractLibraries.TOKEN_MANAGER
        );
        (bool success, ) = tokenAddr.call(
            abi.encodeWithSelector(
                APPROVE_SELECTOR,
                tokenManager,
                type(uint256).max
            )
        );

        if (!success) {
            revert ApproveFailed();
        }
    }
```

Apart from this issue,, there is a distinction in how the approval process is handled for WETH compared to other ERC20 tokens when interacting with the `capitalPool` in `TokenManager` contract
In the provided `TokenManager` contract, there is a distinction in how the approval process is handled for WETH (Wrapped Ether) compared to other ERC20 tokens when interacting with the `capitalPool`. For other ERC20 tokens, the approval process differs. There is no explicit call to the `capitalPool` to approve the transfer amount. Instead, the contract directly attempts to transfer the tokens using the `_safe_transfer_from` function.

## Impact

Given the issues described, there is currently a workaround that users can utilize to withdraw their tokens: they can explicitly call the `capitalPool.approve()` function themselves before attempting to withdraw. By doing this, users can manually grant the necessary allowance for the `TokenManager` to transfer their ERC20 tokens from the `capitalPool`.
However, this workaround is not how the implementation is supposed to function. The intended behavior is for the contract to manage token approvals internally
## Tools Used

Manual review.
## Recommendations

Implement ACL for the `approve()` function in `CapitalPool.sol`.
Add approval for the ERC20 tokens which are going to be transferred like it is done for WETH.


# H04: Incorrect accounting of point tokens for BID Takers

## Summary

When a Maker creates an ASK Offer, they are offering to sell their points at a specified price. Conversely, a Taker might place a BID Order to purchase those points. In this scenario, when the Token Generation Event (TGE) occurs, the Maker is responsible for settling the transaction and providing the tokens. Similarly, the Taker must finalize their bid after the Maker has settled. Once the bid is closed, the Taker should be able to withdraw the points they have acquired. However, due to incorrect accounting logic in the current protocol implementation, Takers are unable to withdraw the points as expected.
## Vulnerability Details
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L187
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L109

The issue stems from the `closeBidTaker()` function within the `DeliveryPlace` contract. Specifically, on line `#187`, the address of the collateral token is mistakenly used instead of the address of the points token. In this scenario, the correct approach should be to account for the number of points tokens being purchased by the Taker, ensuring accurate tracking.

```solidity
tokenManager.addTokenBalance(
    TokenBalanceType.PointToken,
    _msgSender(),
    makerInfo.tokenAddress, // makerInfo.tokenAddress is collateral token, should be point token
    pointTokenAmount
);
```

The following test case can be added to the existing test file:
NOTE: need to add the following line: `import {MarketPlaceStatus} from "../src/interfaces/ISystemConfig.sol";`

```solidity
function testCloseBidTaker() public {
        vm.startPrank(user);
        uint256 initialBalanceUser = mockUSDCToken.balanceOf(user);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Turbo
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        address offer1Addr = GenerateAddress.generateOfferAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        systemConfig.updateMarketPlaceStatus("Backpack", MarketPlaceStatus.AskSettling);
        vm.stopPrank();

        vm.startPrank(user);
        mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
        deliveryPlace.settleAskMaker(offerAddr, 500);
        vm.stopPrank();

        vm.startPrank(user2);
        deliveryPlace.closeBidTaker(stock1Addr);
        // Shows that incorrect amount is accounted for mockUSDCToken, should be 0
        console.log(tokenManager.userTokenBalanceMap(user2, address(mockUSDCToken), TokenBalanceType.PointToken));
    }
```

## Impact

Consequently, Takers are unable to withdraw their points tokens. More critically, the flaw allows users to withdraw collateral tokens instead of the intended points tokens, posing a severe risk to the protocol. This vulnerability could lead to substantial financial losses and significantly disrupt the marketplace's overall functionality.
## Tools Used

Manual review, Foundry.
## Recommendations

Instead of using `makerInfo.tokenAddress` which is the collateral token address, use the address of the points token.

# H05: Collateral fees are stuck when Admin settles ASK Offer instead of Maker

## Summary

When a Maker fails to settle an ASK Offer, the expectation is that the Admin will settle on their behalf, ensuring that the BID Taker receives a portion of the collateral, with the remaining portion allocated to the protocol team. This process relies on accurately accounting for the tokens held by different addresses. However, in the current implementation, when the Admin settles instead of the Maker, the collateral fee is not correctly attributed to the Admin. This results in the funds becoming stuck, as there would be no mechanism for the Admin to withdraw these funds.

## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L257
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L276
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L301

Admin is expected to call `settleAskMaker()` in `DeliveryPlace.sol` instead of Maker.
On line `#257` is checked if input parameter `_settledPoints` is exactly 0 (this is the only value which satisfies the `_settledPoints <= 0` condition, because `_settledPoints` is of unsigned type).
Later, on line `#276` there is another check `_settledPoints == offerInfo.usedPoints` which will be true only if `offerInfo.usedPoints` is 0, which is not the case when there are BID Takers to the ASK Offer.
This leads to skipping the code from line `#301`-`#306` which account for the allowance of token withdrawal for the current msg.sender (in this scenario - Admin).
Because of this inconsistency, Admin wont be able to call `withdraw()` in `TokenManager.sol` later as his accounted allowance would be 0, instead of the remaining collateral.

The following test case shows the described scenario.
NOTE: need to add `deal(user2, 100000000 * 10 ** 18);` in `setUp()`.
```solidity
function testAdminSettleIsNotAccounted() public {
        vm.startPrank(user);

        preMarktes.createOffer{value: 0.012 * 1e18}(
            CreateOfferParams(
                marketPlace, address(weth9), 1000, 0.01 * 1e18, 12000, 300, OfferType.Ask, OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker{value: 0.005175 * 1e18}(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        // Admin settles instead of Maker after settlement period is finished
        // warp to set timestamp after settlement period is finished
        vm.warp(1719826111275);
        deliveryPlace.settleAskMaker(offerAddr, 0);
        vm.stopPrank();

        vm.startPrank(user2);
        mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
        deliveryPlace.closeBidTaker(stock1Addr);

        // Taker gets his part of the collateral correctly.
        console.log(tokenManager.userTokenBalanceMap(address(user2), address(address(weth9)), TokenBalanceType.RemainingCash));
        
        // Admin is not accounted correctly and balances remain 0.
        console.log(tokenManager.userTokenBalanceMap(address(user1), address(weth9), TokenBalanceType.MakerRefund));
        console.log(tokenManager.userTokenBalanceMap(address(user1), address(weth9), TokenBalanceType.SalesRevenue));
        vm.stopPrank();
    }
```
## Impact

Collateral of Maker who do not settle remains stuck in the protocol due to wrong accounting.
## Tools Used

Manual review, Foundry.
## Recommendations

Consider accounting for the collateral fee, when settling ASK Offers as Admin instead of the Maker who created the Offer.

# H06: Collateral fees are stuck when Admin settles ASK Order instead of Taker

## Summary

When the Admin settles on behalf of an ASK Taker, who is responsible for providing tokens after placing a BID Offer, the collateral provided by the Taker becomes stuck in the protocol. This happens because the current implementation lacks a mechanism for the Admin to withdraw the collateral, leaving the funds trapped.
## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L335
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L368
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L376
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L400


Admin needs to call `settleAskTaker()` in `DeliveryPlace.sol` so the BID Maker is able to call `closeBidOffer()` in the same contract. Logic in the `settleAskTaker()` does not account the collateral provided by the ASK Taker.

On line `#368`, the condition `if (_settledPoints > 0)` will always evaluate to false because the only possible value for `_settledPoints` is 0. This causes the subsequent conditions on line `#376` (`if (settledPointTokenAmount > 0)`) and line `#400` (`if (_settledPoints == stockInfo.points)`) to also fail. As a result, the protocol fails to account for the collateral that the Admin should receive from the ASK Taker, leading to unaccounted and stuck funds.

The following test case shows the described scenario:

```solidity
    function testAdminSettleAskTaker() public {
        vm.startPrank(user);
        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Bid,
                OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        // Admin settles instead of ASK Taker after settlement period is finished
        // warp to set timestamp after settlement period is finished
        vm.warp(1719826111275);
        deliveryPlace.settleAskTaker(stock1Addr, 0);
        vm.stopPrank();

        vm.startPrank(user);
        deliveryPlace.closeBidOffer(offerAddr);
        // BID Maker has received back his collateral + fee
        console.log(tokenManager.userTokenBalanceMap(address(user), address(mockUSDCToken), TokenBalanceType.MakerRefund));
        // Admin is not accounted for the collateral provided by ASK Taker.
console.log(tokenManager.userTokenBalanceMap(address(user1), address(mockUSDCToken), TokenBalanceType.RemainingCash));
        vm.stopPrank();
}
```
## Impact

Funds stuck in the contract due to wrong accounting.
## Tools Used

Manual review, Foundry.
## Recommendations

Consider adding logic to ensure that when the Admin settles, the Admin is properly accounted for and can withdraw the funds provided by the party that failed to settle (in this case, the ASK Taker).
# M01: Makers can abort ASK Offers in TURBO mode even after takers have placed orders and listed their acquired stock

## Summary

In Turbo mode, the initial offer maker is responsible for settling all subsequent trades that stem from their ASK offer. Consequently, the maker should be prevented from canceling the ASK offer if there are any subsequent listings based on the stock originally acquired from it. Permitting the maker to abort the ASK offer under these circumstances would disrupt the continuity of trades, potentially leaving transactions unsettled. However, the current implementation fails to correctly track whether the initial offer has such subsequent listings, leading to potential disruptions in the trading process.

## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L342
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L552

Line `#342` in `listOffer()` function inside `PreMarkets.sol` is intended to change the `abortOfferStatus` of the initial offer and this variable is later checked in on line `#552` in `abortAskOffer()` in the same file. |
Saving `OfferInfo` in memory with `OfferInfo memory originOfferInfo = offerInfoMap[originOffer];` and then modifying it with `originOfferInfo.abortOfferStatus = AbortOfferStatus.SubOfferListed;` only updates the local memory copy, not the original data in `offerInfoMap`. This leads to an inconsistent state, as the changes do not persist in the contract's storage.

```solidity
...
if (makerInfo.offerSettleType == OfferSettleType.Turbo) {
            address originOffer = makerInfo.originOffer;
            // variable saved in memory, instead of storage
            OfferInfo memory originOfferInfo = offerInfoMap[originOffer];

            if (_collateralRate != originOfferInfo.collateralRate) {
                revert InvalidCollateralRate();
            }
            // changing value of abortOfferStatus, wouldnt change the state variable
            originOfferInfo.abortOfferStatus = AbortOfferStatus.SubOfferListed;
}
...
        

```

```solidity
function abortAskOffer(address _stock, address _offer) external {
        StockInfo storage stockInfo = stockInfoMap[_stock];
        OfferInfo storage offerInfo = offerInfoMap[_offer];

        if (offerInfo.authority != _msgSender()) {
            revert Errors.Unauthorized();
        }

        if (stockInfo.offer != _offer) {
            revert InvalidOfferAccount(stockInfo.offer, _offer);
        }

        if (offerInfo.offerType != OfferType.Ask) {
            revert InvalidOfferType(OfferType.Ask, offerInfo.offerType);
        }

		// Check should prevent of aborting offers which have subsequent listing of stocks acquired from this offer.
        if (offerInfo.abortOfferStatus != AbortOfferStatus.Initialized) {
            revert InvalidAbortOfferStatus(
                AbortOfferStatus.Initialized,
                offerInfo.abortOfferStatus
            );
        }
        ...
}
```

The following test shows a scenario, where initial Maker can abort the offer after subsequent listing.

```solidity
function testAbortAskOfferAfterSubsequentListing() public {
        vm.startPrank(user);

        preMarktes.createOffer(
            CreateOfferParams(
                marketPlace,
                address(mockUSDCToken),
                1000,
                0.01 * 1e18,
                12000,
                300,
                OfferType.Ask,
                OfferSettleType.Turbo
            )
        );
        vm.stopPrank();

        vm.startPrank(user1);
        mockUSDCToken.approve(address(tokenManager), type(uint256).max);
        address stockAddr = GenerateAddress.generateStockAddress(0);
        address stockAddr1 = GenerateAddress.generateStockAddress(1);
        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address offerAddr1 = GenerateAddress.generateOfferAddress(1);

        preMarktes.createTaker(offerAddr, 500);
        preMarktes.listOffer(stockAddr1, 500, 12000);
        vm.stopPrank();

        vm.startPrank(user3);
        mockUSDCToken.approve(address(tokenManager), type(uint256).max);
        preMarktes.createTaker(offerAddr1, 500);
        vm.stopPrank();

        vm.prank(user);
        preMarktes.abortAskOffer(stockAddr, offerAddr);
    }
```
## Impact

While this issue does not cause direct financial losses for the protocol or its users, it can indirectly harm potential traders who may be unable to purchase the points they intended. This could lead to missed opportunities.
## Tools Used

Manual review, Foundry.
## Recommendations

Update the `abortOfferStatus` directly in the `offerInfoMap` storage rather than in memory.
`OfferInfo storage originOfferInfo = offerInfoMap[originOffer];`


# M02: User funds will be stuck if maker does not settle and admin key is compromised/lost

## Summary

If an ASK offer Maker fails to settle, the BID Taker should be entitled to receive a portion of the collateral provided by the Maker. However, the current implementation relies entirely on the Admin to settle on behalf of the Maker, meaning the Taker cannot withdraw their funds until the Admin intervenes. While the Admin role is trusted, this approach poses a risk. Implementing a mechanism that allows users to withdraw their funds automatically after a certain period would enhance the trustless nature of the system and mitigate risks associated with a compromised Admin.

## Vulnerability Details
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L222
https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/DeliveryPlace.sol#L96

If Makers do not settle, it is expected that `settleAskMaker()` in `DeliveryPlace.sol` will be called by the Admin in order for the Taker to be able to call `closeBidTaker()` in the same file.

Following test case shows the expected workflow:
NOTE: need to add `deal(user2, 100000000 * 10 ** 18);` in `setUp()
`
```solidity
    function testShowAdminSettle() public {
        vm.startPrank(user);

        preMarktes.createOffer{value: 0.012 * 1e18}(
            CreateOfferParams(
                marketPlace, address(weth9), 1000, 0.01 * 1e18, 12000, 300, OfferType.Ask, OfferSettleType.Protected
            )
        );

        address offerAddr = GenerateAddress.generateOfferAddress(0);
        address stock1Addr = GenerateAddress.generateStockAddress(1);
        vm.stopPrank();

        vm.prank(user2);
        preMarktes.createTaker{value: 0.005175 * 1e18}(offerAddr, 500);

        vm.startPrank(user1);
        systemConfig.updateMarket("Backpack", address(mockPointToken), 0.01 * 1e18, block.timestamp - 1, 3600);
        // Admin settles instead of Maker after settlement period is finished
        // warp to set timestamp after settlement period is finished
        vm.warp(1719826111275);
        deliveryPlace.settleAskMaker(offerAddr, 0);
        vm.stopPrank();

        vm.startPrank(user2);
        mockPointToken.approve(address(tokenManager), 10000 * 10 ** 18);
        deliveryPlace.closeBidTaker(stock1Addr);
        vm.stopPrank();
    }
```

## Impact

If the Maker does not settle on time, user funds are entirely dependent on the Admin to settle. In the event of compromised or lost Admin keys, user funds could be permanently locked. This approach undermines the principles of decentralization and a trustless system.
## Tools Used

Manual review, Foundry.
## Recommendations

Consider implementing a mechanism that allows Takers to withdraw their funds without relying on the Admin if they don't receive their point tokens. For example, you could add a check inside `closeBidTaker()` to verify if the settlement period has expired and the Maker has not settled. If so, Takers should be allowed to withdraw their funds automatically after that period. This would ensure that users are not left waiting indefinitely for settlement and enhance the protocol's decentralization and trustlessness.



# L01: Wrong event emitted when calling settleAskTaker()

## Summary

Incorrect event is emitted when calling `settleAskTaker()`.
Instead of emitting `SettledAskTaker`, `SettledBidTaker` is emitted.

## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L775

On line `#775` in `PreMarkets.sol` `SettledBidTaker` is emitted instead of `SettledAskTaker`.
## Impact

Wrong event emitted can lead to inconsistency in off-chain related software.
## Tools Used

Manual review.
## Recommendations

Emit `SettledAskTaker` instead of `SettledBidTaker`.


# L02: Fee on transfer ERC20 collateral tokens can lead to DoS when trying to transfer them

## Summary

Checking for balance before and after transferring tokens is incompatible with ERC20 tokens that implement fee-on-transfer functionality. This is because fee-on-transfer tokens deduct a fee during the transfer process, causing the post-transfer balance to be less than expected. As a result, simple balance checks may incorrectly flag successful transfers as failures, leading to issues in handling such tokens correctly.
## Vulnerability Details

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/TokenManager.sol#L233

`_transfer()` function in `TokenManager.sol` 
```solidity
function _transfer(
        address _token,
        address _from,
        address _to,
        uint256 _amount,
        address _capitalPoolAddr
    ) internal {
        uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
        uint256 toBalanceBef = IERC20(_token).balanceOf(_to);

        if (
            _from == _capitalPoolAddr &&
            IERC20(_token).allowance(_from, address(this)) == 0x0
        ) {
            ICapitalPool(_capitalPoolAddr).approve(address(this));
        }

        _safe_transfer_from(_token, _from, _to, _amount);

        uint256 fromBalanceAft = IERC20(_token).balanceOf(_from);
        uint256 toBalanceAft = IERC20(_token).balanceOf(_to);

        if (fromBalanceAft != fromBalanceBef - _amount) {
            revert TransferFailed();
        }

        if (toBalanceAft != toBalanceBef + _amount) {
            revert TransferFailed();
        }
    }
}
```

## Impact

Some tokens can lead to DoS of the protocol functionality.
## Tools Used

Manual review.
## Recommendations

Instead of strictly comparing `fromBalanceAft` with `fromBalanceBef - _amount`, calculate the actual amount transferred by subtracting `fromBalanceAft` from `fromBalanceBef`. This allows the function to accommodate fee-on-transfer tokens, where the `actualAmountTransferred` might be less than the `_amount`.

Based on the protocol team, accepted fee can be adjusted.
