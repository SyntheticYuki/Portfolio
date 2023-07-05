![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2F512_black.png&w=256&q=75)

# [Symmetrical](https://audits.sherlock.xyz/contests/85)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Symmetrical | [Website](https://www.symm.io) | [Twitter](https://twitter.com/symm_io) | 61,000 USDC | 3121 | 18 days | June 15, 2023 | July 3, 2023 |


## What is SYMMIO?

- SYMMIO is a dedicated protocol devised for trading Symmetrical Derivatives. Recognized as a derivatives protocol suite, SYMMIO is a platform that orchestrates the standard for onchain communications utilized for Symmetrical Trading, leveraging automated Markets for Quotes (AMFQ), a request-based trading framework for onchain derivatives.
The foundations of the suite comprise the Ethereum protocol, the SYMMIO protocol, and the Frontend SDK.  The SYMM IO protocol suite furnishes the tools for enabling AMFQ onchain communication, precisely defining how quotes should be structured, requested, and accepted.
With its unique features and capabilities, SYMM IO is poised to become the go-to protocol for onchain derivatives trading

## Audit scope

- [symmio-core/contracts/Diamond.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/Diamond.sol)
- [symmio-core/contracts/facets/DiamondCutFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/DiamondCutFacet.sol)
- [symmio-core/contracts/facets/DiamondLoupFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/DiamondLoupFacet.sol)
- [symmio-core/contracts/facets/Account/AccountFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacet.sol)
- [symmio-core/contracts/facets/Account/AccountFacetImpl.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacetImpl.sol)
- [symmio-core/contracts/facets/PartyA/PartyAFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyA/PartyAFacet.sol)
- [symmio-core/contracts/facets/PartyA/PartyAFacetImpl.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyA/PartyAFacetImpl.sol)
- [symmio-core/contracts/facets/PartyB/PartyBFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacet.sol)
- [symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol)
- [symmio-core/contracts/facets/control/ControlFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/control/ControlFacet.sol)
- [symmio-core/contracts/facets/liquidation/LiquidationFacet.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacet.sol)
- [symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol](https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol)

## Issues found by Yuki

| Severity | Title | Count |
|:--|:--|:--:|
| High | Wrong accounting happens when opening a partially filled position, which leads to permanent loss of funds. | 1 |
| High | Malicious Party B is able to permanently prevent force closing a position by partially closing dust amounts. | 2 |
| High | Expired signature can stuck all Party positions in a liquidation state. | 3 |
| High | Loss of funds duo to wrong accounting of decimals when changing collateral. | 4 |
| Medium | Malicious Party A can prevent Party B from emergency closing a position on a market price. | 5 |
| Medium | Malicious liquidator can get the liquidation fee without finalizing the full liquidation of Party A. | 6 |
| Medium | Force closing a position won't work when the order type is MARKET. | 7 |

# 1. Wrong accounting happens when opening a partially filled position, which leads to permanent loss of funds.

## Summary
Wrong accounting happens when opening a partially filled position, which leads to permanent loss of funds.

## Vulnerability Detail
The process of opening a position is simple, if the order is MARKET the user will need to provide the full amount of the Quote quantity in order to open the position. On the other hand if the order is LIMIT, the user can provide an amount they want up to the full quantity of the Quote.

Depending on the circumstances, when a LIMIT order is partially filled. The function can create a new PENDING Quote with the leftover locked values.

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L112-#L254

The issue occurs, when a order is partially filled and a new pending Quote is issued.

The else statement below is triggered when the newStatus of the Quote is PENDING, the function subtracts the filledLockedValues from the pendingLockedBalances, but makes the huge mistake to do the same for the partyBPendingLockedBalances.

```solidity
            if (newStatus == QuoteStatus.CANCELED) {
                // send trading Fee back to partyA
                LibQuote.returnTradingFee(currentId);
                // part of quote has been filled and part of it has been canceled
                accountLayout.pendingLockedBalances[quote.partyA].subQuote(quote);
                accountLayout.partyBPendingLockedBalances[quote.partyB][quote.partyA].subQuote(
                    quote
                );
            } else {
                accountLayout.pendingLockedBalances[quote.partyA].sub(filledLockedValues);
                accountLayout.partyBPendingLockedBalances[quote.partyB][quote.partyA].sub(
                    filledLockedValues
                );
            }
```

The new Quote created is PENDING and PartyB is reset to the zero address.

As a result the old Party B no longer needs to hold the leftover locked values in his partyBPendingLockedBalances.
This logic error is permanent and can't be fixed as the new Quote with the leftover locked values is PENDING.
Party B can't do nothing in order to get rid of the wrongly accounted leftover locked values in his partyBPendingLockedBalances mapping.

```solidity
            Quote memory q = Quote({
                id: currentId,
                partyBsWhiteList: quote.partyBsWhiteList,
                symbolId: quote.symbolId,
                positionType: quote.positionType,
                orderType: quote.orderType,
                openedPrice: 0,
                requestedOpenPrice: quote.requestedOpenPrice,
                marketPrice: quote.marketPrice,
                quantity: quote.quantity - filledAmount,
                closedAmount: 0,
                lockedValues: LockedValues(0, 0, 0),
                initialLockedValues: LockedValues(0, 0, 0),
                maxInterestRate: quote.maxInterestRate,
                partyA: quote.partyA,
                partyB: address(0),
                quoteStatus: newStatus,
                avgClosedPrice: 0,
                requestedClosePrice: 0,
                parentId: quote.id,
                createTimestamp: quote.createTimestamp,
                modifyTimestamp: block.timestamp,
                quantityToClose: 0,
                deadline: quote.deadline
            });
```

Duo to the wrong accounting of the partyBPendingLockedbalances, this logic error leads to permanent loss of funds.
As PartyB won't be able to deallocate his full balance of funds in order to withdraw them.

```solidity
    function deallocateForPartyB(
        uint256 amount,
        address partyA,
        SingleUpnlSig memory upnlSig
    ) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        require(
            accountLayout.partyBAllocatedBalances[msg.sender][partyA] >= amount,
            "PartyBFacet: Insufficient locked balance"
        );
        LibMuon.verifyPartyBUpnl(upnlSig, msg.sender, partyA);
        int256 availableBalance = LibAccount.partyBAvailableForQuote(
            upnlSig.upnl,
            msg.sender,
            partyA
        );
        require(availableBalance >= 0, "PartyBFacet: Available balance is lower than zero");
        require(uint256(availableBalance) >= amount, "PartyBFacet: Will be liquidatable");

        accountLayout.partyBNonces[msg.sender][partyA] += 1;
        accountLayout.partyBAllocatedBalances[msg.sender][partyA] -= amount;
        accountLayout.balances[msg.sender] += amount;
        accountLayout.withdrawCooldown[msg.sender] = block.timestamp;
    }
```
```solidity
    function partyBAvailableForQuote(
        int256 upnl,
        address partyB,
        address partyA
    ) internal view returns (int256) {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        int256 available;
        if (upnl >= 0) {
            available =
                int256(accountLayout.partyBAllocatedBalances[partyB][partyA]) +
                upnl -
                int256(
                    (accountLayout.partyBLockedBalances[partyB][partyA].total() +
                        accountLayout.partyBPendingLockedBalances[partyB][partyA].total())
                );
        } else {
            int256 mm = int256(accountLayout.partyBLockedBalances[partyB][partyA].mm);
            int256 considering_mm = -upnl > mm ? -upnl : mm;
            available =
                int256(accountLayout.partyBAllocatedBalances[partyB][partyA]) -
                int256(
                    (accountLayout.partyBLockedBalances[partyB][partyA].cva +
                        accountLayout.partyBLockedBalances[partyB][partyA].lf +
                        accountLayout.partyBPendingLockedBalances[partyB][partyA].total())
                ) -
                considering_mm;
        }
        return available;
    }
```

## Impact
Partially filling an order leads to the wrong accounting of PartyB balances, when new pending Quote is issued.
On the other hand this wrong accounting leads to permanent loss of funds as Party B won't be able to deallocate his full balance of funds.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L239-L241

## Tool used

Manual Review

## Recommendation
When creating a new pending Quote, the whole old quote values should be subtracted from partyBPendingLockedBalances. As PartyB doesn't need to hold the leftover locked values for the new PENDING Quote.

```solidity
            if (newStatus == QuoteStatus.CANCELED) {
                // send trading Fee back to partyA
                LibQuote.returnTradingFee(currentId);
                // part of quote has been filled and part of it has been canceled
                accountLayout.pendingLockedBalances[quote.partyA].subQuote(quote);
                accountLayout.partyBPendingLockedBalances[quote.partyB][quote.partyA].subQuote(
                    quote
                );
            } else {
                accountLayout.pendingLockedBalances[quote.partyA].sub(filledLockedValues);
                accountLayout.partyBPendingLockedBalances[quote.partyB][quote.partyA].sub(
                   quote
                );
            }
```

# 2. Malicious Party B is able to permanently prevent force closing a position by partially closing dust amounts.

## Summary
Malicious Party B is able to prevent force closing a position by partially closing dust amounts.

## Vulnerability Detail
The function forceClosePosition was made incase a Party A request to close a position, but Party B is malicious and doesn't respond. In the end after the force time cooldown runs out, the function forceClosePosition can be called to forcely close the given amount of quantity from the open position.

However this can be easy exploited by Party B, if we look over the function forceClosePosition we can see that one of the main requirements is for the cooldown to be reached which is mainly based on the quote.modifyTimestamp while the other important requirement is for the quote to be a LIMIT order.

MARKET - requires all of the quantity to be closed
LIMIT - can be partially closed

Since the MARKET order is restricted from force closing the position, the only way would be to set the order type as LIMIT which is exploitable by Malicious Party B.

- PartyA requests MARKET order close

- PartyB doesn't respond (malicious)

- Market close expires (can't be force closed as a position)

- PartyA requests a LIMIT order close

- PartyB (malicious) partially closes dust amounts in order to reset the quote.modifyTimestamp to the current block.timestamp

In the end Party A doesn't have a way to close the position.

```solidity
    function forceClosePosition(uint256 quoteId, PairUpnlAndPriceSig memory upnlSig) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        MAStorage.Layout storage maLayout = MAStorage.layout();
        Quote storage quote = QuoteStorage.layout().quotes[quoteId];

        uint256 filledAmount = quote.quantityToClose;
        require(quote.quoteStatus == QuoteStatus.CLOSE_PENDING, "PartyAFacet: Invalid state");
        require(
            block.timestamp > quote.modifyTimestamp + maLayout.forceCloseCooldown,
            "PartyAFacet: Cooldown not reached"
        );
        require(block.timestamp <= quote.deadline, "PartyBFacet: Quote is expired");
        require(
            quote.orderType == OrderType.LIMIT,
            "PartyBFacet: Quote's order type should be LIMIT"
        );
        if (quote.positionType == PositionType.LONG) {
            require(
                upnlSig.price >=
                    quote.requestedClosePrice +
                        (quote.requestedClosePrice * maLayout.forceCloseGapRatio) /
                        1e18,
                "PartyAFacet: Requested close price not reached"
            );
        } else {
            require(
                upnlSig.price <=
                    quote.requestedClosePrice -
                        (quote.requestedClosePrice * maLayout.forceCloseGapRatio) /
                        1e18,
                "PartyAFacet: Requested close price not reached"
            );
        }

        LibMuon.verifyPairUpnlAndPrice(upnlSig, quote.partyB, quote.partyA, quote.symbolId);
        LibSolvency.isSolventAfterClosePosition(
            quoteId,
            filledAmount,
            quote.requestedClosePrice,
            upnlSig
        );
        accountLayout.partyANonces[quote.partyA] += 1;
        accountLayout.partyBNonces[quote.partyB][quote.partyA] += 1;
        LibQuote.closeQuote(quote, filledAmount, quote.requestedClosePrice);
    }
```
The first thing the function does when closing a Quote is to update the quote.modifyTimestamp to the current block.timestamp. Therefore Malicious Party B is able to permanently prevent force closing a position by partially closing dust amounts.

```solidity
    function closeQuote(Quote storage quote, uint256 filledAmount, uint256 closedPrice) internal {
        QuoteStorage.Layout storage quoteLayout = QuoteStorage.layout();
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();

        quote.modifyTimestamp = block.timestamp;

        LockedValues memory lockedValues = LockedValues(
            quote.lockedValues.cva -
                ((quote.lockedValues.cva * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
            quote.lockedValues.mm -
                ((quote.lockedValues.mm * filledAmount) / (LibQuote.quoteOpenAmount(quote))),
            quote.lockedValues.lf -
                ((quote.lockedValues.lf * filledAmount) / (LibQuote.quoteOpenAmount(quote)))
        );
        accountLayout.lockedBalances[quote.partyA].subQuote(quote).add(lockedValues);
        accountLayout.partyBLockedBalances[quote.partyB][quote.partyA].subQuote(quote).add(
            lockedValues
        );
        quote.lockedValues = lockedValues;

        (bool hasMadeProfit, uint256 pnl) = LibQuote.getValueOfQuoteForPartyA(
            closedPrice,
            filledAmount,
            quote
        );
        if (hasMadeProfit) {
            accountLayout.allocatedBalances[quote.partyA] += pnl;
            accountLayout.partyBAllocatedBalances[quote.partyB][quote.partyA] -= pnl;
        } else {
            accountLayout.allocatedBalances[quote.partyA] -= pnl;
            accountLayout.partyBAllocatedBalances[quote.partyB][quote.partyA] += pnl;
        }

        quote.avgClosedPrice =
            (quote.avgClosedPrice * quote.closedAmount + filledAmount * closedPrice) /
            (quote.closedAmount + filledAmount);

        quote.closedAmount += filledAmount;
        quote.quantityToClose -= filledAmount;

        if (quote.closedAmount == quote.quantity) {
            quote.quoteStatus = QuoteStatus.CLOSED;
            quote.requestedClosePrice = 0;
            removeFromOpenPositions(quote.id);
            quoteLayout.partyAPositionsCount[quote.partyA] -= 1;
            quoteLayout.partyBPositionsCount[quote.partyB][quote.partyA] -= 1;
        } else if (
            quote.quoteStatus == QuoteStatus.CANCEL_CLOSE_PENDING || quote.quantityToClose == 0
        ) {
            quote.quoteStatus = QuoteStatus.OPENED;
            quote.requestedClosePrice = 0;
            quote.quantityToClose = 0; // for CANCEL_CLOSE_PENDING status
        } else {
            require(
                quote.lockedValues.total() >=
                    SymbolStorage.layout().symbols[quote.symbolId].minAcceptableQuoteValue,
                "LibQuote: Remaining quote value is low"
            );
        }
```

## Impact
A malicious Party B is able to permanently prevent a force closing a position by partially closing dust amounts to reset the quote.modifyTimestamp.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyA/PartyAFacetImpl.sol#L253

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/libraries/LibQuote.sol#L153

## Tool used

Manual Review

## Recommendation
A mofifyTimestamp of a quote should be updated only when the quantityToClose equals zero, otherwise Party B can take advantage of this issue.

```solidity
        if (quote.closedAmount == quote.quantity) {
            quote.quoteStatus = QuoteStatus.CLOSED;
            quote.requestedClosePrice = 0;
            removeFromOpenPositions(quote.id);
            quoteLayout.partyAPositionsCount[quote.partyA] -= 1;
            quoteLayout.partyBPositionsCount[quote.partyB][quote.partyA] -= 1;
        } else if (
            quote.quoteStatus == QuoteStatus.CANCEL_CLOSE_PENDING || quote.quantityToClose == 0
        ) {
            quote.quoteStatus = QuoteStatus.OPENED;
            quote.requestedClosePrice = 0;
            quote.quantityToClose = 0; // for CANCEL_CLOSE_PENDING status
+           quote.modifyTimestamp = block.timestamp;
        } else {
            require(
                quote.lockedValues.total() >=
                    SymbolStorage.layout().symbols[quote.symbolId].minAcceptableQuoteValue,
                "LibQuote: Remaining quote value is low"
            );
        }
    }
```
# 3. Expired signature can stuck all Party positions in a liquidation state.

## Summary
Expired signature can stuck all Party positions in a liquidation state.

## Vulnerability Detail
If we look over the liquidation path of Party A when setting the symbol prices, we can see that there is a liquidation timer before the signature expires.

This is a problem because once the signature expires, there is now way to set the symbol price for the given quote and therefore the liquidation of Party A can't be finished. On the other part the function liquidatePartyA can't be called a second time to reset the liquidation timestamp, because Party A already has the liquidation status.

Path to successfully finish the liquidation of Party A:

liquidatePartyA -> setSymbolPrices -> liquidatePendingPositionsPartyA -> liquidatePositionsPartyA

Which means that Party A positions are permanently stuck on the user's side. And until the liquidation is complete Party A can't use most of the protocol duo to the liquidation status.

```solidity
    function setSymbolsPrice(address partyA, PriceSig memory priceSig) internal {
        MAStorage.Layout storage maLayout = MAStorage.layout();
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();

        LibMuon.verifyPrices(priceSig, partyA);
        require(maLayout.liquidationStatus[partyA], "LiquidationFacet: PartyA is solvent");
        require(
            priceSig.timestamp <=
                maLayout.liquidationTimestamp[partyA] + maLayout.liquidationTimeout,
            "LiquidationFacet: Expired signature"
        );
```
Incase this issue occurs, the protocol team will need to manually change the liquidationTimestamp for every expired signature there is in order to fix the issue and unstuck the liquidation.

Note - the issue also occurs when liquidating Party B.

## Impact
Expired signature permanently DoS Party A liquidation on the user's side.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L34

## Tool used

Manual Review

## Recommendation
There should be a way to reset the liquidation timestamp of a Party on the user's side. Instead of the need to manually change the liquidation timeout for every expired signature there is.

# 4. Loss of funds duo to wrong accounting of decimals when changing collateral.

## Summary
Changing the collateral leads to the wrong accounting of the existing balances duo to decimals difference between USDC and USDT.

## Vulnerability Detail
On the sherlock contest page, it is shown the symmetrical protocol can interact with both USDC and USDT tokens. Which has different decimals on the most chains: USDC - 18 decimals, USDT - 6 decimals.

<img width="611" alt="Screenshot 2023-07-04 at 13 49 17" src="https://github.com/SilentYuki/Portfolio/assets/135425690/b99ab9db-2cd3-4116-a022-8d07b2545121">

By looking at the protocol design, the amount which is added to the users balances is calculated in 18 decimals, so everything will work fine and the decimals of the tokens won't matter here.

```solidity
    function deposit(address user, uint256 amount) internal {
        GlobalAppStorage.Layout storage appLayout = GlobalAppStorage.layout();
        IERC20(appLayout.collateral).safeTransferFrom(msg.sender, address(this), amount);
        uint256 amountWith18Decimals = (amount * 1e18) /
        (10 ** IERC20Metadata(appLayout.collateral).decimals());
        AccountStorage.layout().balances[user] += amountWith18Decimals;
    }
```
```solidity
    function withdraw(address user, uint256 amount) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        GlobalAppStorage.Layout storage appLayout = GlobalAppStorage.layout();
        require(
            block.timestamp >=
            accountLayout.withdrawCooldown[msg.sender] + MAStorage.layout().deallocateCooldown,
            "AccountFacet: Cooldown hasn't reached"
        );
        uint256 amountWith18Decimals = (amount * 1e18) /
        (10 ** IERC20Metadata(appLayout.collateral).decimals());
        accountLayout.balances[msg.sender] -= amountWith18Decimals;
        IERC20(appLayout.collateral).safeTransfer(user, amount);
    }
```
The problem occurs when the protocol team wants to change the collateral between USDC or USDT.
This will lead to the wrong accounting of the existing balances, as they will no longer be calculated based on the old decimals but the new ones.

```solidity
    function setCollateral(
        address collateral
    ) external onlyRole(LibAccessibility.DEFAULT_ADMIN_ROLE) {
        GlobalAppStorage.layout().collateral = collateral;
        emit SetCollateral(collateral);
    }
```

## Impact
Wrong accounting of the existing balances duo to decimal difference.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/Account/AccountFacetImpl.sol#L27

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/control/ControlFacet.sol#L95

## Tool used

Manual Review

## Recommendation
As how it is right now, the protocol design doesn't allow changing the collateral when there are existing balances.
The code should be changed or the function setCollateral should be removed.

# 5. Malicious Party A can prevent Party B from emergency closing a position on a market price.

## Summary
Malicious Party A who doesn't want the position to be closed on a market price can temporary DoS Party B from closing the position in emergency mode.

## Vulnerability Detail
When the protocol is in emergency mode or the user has the emergency status given to him. He is able to emergency close a position on the market price.

Currently in order to emergency close a position the quote needs to be with the status "OPENED".
This is exploitable incase Party A is malicious and doesn't want the position to be closed at a market price.

```solidity
    function emergencyClosePosition(uint256 quoteId, PairUpnlAndPriceSig memory upnlSig) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        Quote storage quote = QuoteStorage.layout().quotes[quoteId];
        require(quote.quoteStatus == QuoteStatus.OPENED, "PartyBFacet: Invalid state");
        LibMuon.verifyPairUpnlAndPrice(upnlSig, quote.partyB, quote.partyA, quote.symbolId);
        uint256 filledAmount = LibQuote.quoteOpenAmount(quote);
        quote.quantityToClose = filledAmount;
        quote.requestedClosePrice = upnlSig.price;
        LibSolvency.isSolventAfterClosePosition(quoteId, filledAmount, upnlSig.price, upnlSig);
        accountLayout.partyBNonces[quote.partyB][quote.partyA] += 1;
        accountLayout.partyANonces[quote.partyA] += 1;
        LibQuote.closeQuote(quote, filledAmount, upnlSig.price);
    }
```
Party A can front run Party B by calling requestToClosePosition and setting the desired close price they want and a long deadLine time. After the call the quote will have a CLOSE_PENDING status and Party B won't have other choice except to wait for the long deadLine to pass.

Conclusion:
Malicious Party A are able to temporary DoS Party B from closing a position in emergency mode.

```solidity
    function requestToClosePosition(
        uint256 quoteId,
        uint256 closePrice,
        uint256 quantityToClose,
        OrderType orderType,
        uint256 deadline,
        SingleUpnlAndPriceSig memory upnlSig
    ) internal {
        SymbolStorage.Layout storage symbolLayout = SymbolStorage.layout();
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        Quote storage quote = QuoteStorage.layout().quotes[quoteId];

        require(quote.quoteStatus == QuoteStatus.OPENED, "PartyAFacet: Invalid state");
        require(deadline >= block.timestamp, "PartyAFacet: Low deadline");
        require(
            LibQuote.quoteOpenAmount(quote) >= quantityToClose,
            "PartyAFacet: Invalid quantityToClose"
        );
        LibMuon.verifyPartyAUpnlAndPrice(upnlSig, quote.partyA, quote.symbolId);
        LibSolvency.isSolventAfterRequestToClosePosition(
            quoteId,
            closePrice,
            quantityToClose,
            upnlSig
        );

        // check that remaining position is not too small
        if (LibQuote.quoteOpenAmount(quote) > quantityToClose) {
            require(
                ((LibQuote.quoteOpenAmount(quote) - quantityToClose) * quote.lockedValues.total()) /
                    LibQuote.quoteOpenAmount(quote) >=
                    symbolLayout.symbols[quote.symbolId].minAcceptableQuoteValue,
                "PartyAFacet: Remaining quote value is low"
            );
        }

        accountLayout.partyANonces[quote.partyA] += 1;
        quote.modifyTimestamp = block.timestamp;
        quote.quoteStatus = QuoteStatus.CLOSE_PENDING;
        quote.requestedClosePrice = closePrice;
        quote.quantityToClose = quantityToClose;
        quote.orderType = orderType;
        quote.deadline = deadline;
    }
```

## Impact
Malicious Party A is able to temporary DoS Party B from emergency closing a position.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyB/PartyBFacetImpl.sol#L309

## Tool used

Manual Review

## Recommendation
Malicious Parties should not be able to prevent closing a position on market price in emergency mode.

The function emergencyClosePosition should be callable when the quote status is in CLOSE_PENDING as well:

```solidity
    function emergencyClosePosition(uint256 quoteId, PairUpnlAndPriceSig memory upnlSig) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        Quote storage quote = QuoteStorage.layout().quotes[quoteId];
        require(quote.quoteStatus == QuoteStatus.OPENED || quote.quoteStatus == QuoteStatus.CLOSE_PENDING , "PartyBFacet: Invalid state");
        LibMuon.verifyPairUpnlAndPrice(upnlSig, quote.partyB, quote.partyA, quote.symbolId);
        uint256 filledAmount = LibQuote.quoteOpenAmount(quote);
        quote.quantityToClose = filledAmount;
        quote.requestedClosePrice = upnlSig.price;
        LibSolvency.isSolventAfterClosePosition(quoteId, filledAmount, upnlSig.price, upnlSig);
        accountLayout.partyBNonces[quote.partyB][quote.partyA] += 1;
        accountLayout.partyANonces[quote.partyA] += 1;
        LibQuote.closeQuote(quote, filledAmount, upnlSig.price);
    }

}
```

# 6. Malicious liquidator can get the liquidation fee without finalizing the full liquidation of Party A.

## Summary
Malicious liquidator can get the liquidation fee without finalizing the full liquidation of Party A.

## Vulnerability Detail
If we look at the sherlock contest page, it is clearly stated that every role is trusted except the liquidator role.
<img width="787" alt="Screenshot 2023-07-04 at 13 56 58" src="https://github.com/SilentYuki/Portfolio/assets/135425690/7a9a806c-bc4a-472d-9998-2bb90890f3bf">

On the current protocol design, it is possible that a malicious liquidator can receive the liquidation fee without actually finishing the full liquidation of Party A.

The path for successfully liquidating Party A is:

liquidatePartyA -> setSymbolPrices -> liquidatePendingPositionsPartyA -> liquidatePositionsPartyA

The liquidation fee is paid to the addresses from the array liquidatorsPartyA[] in the final function liquidatePositionsPartyA, when all the positions are liquidated.

```solidity
            if (lf > 0) {
                accountLayout.allocatedBalances[accountLayout.liquidators[partyA][0]] += lf / 2;
                accountLayout.allocatedBalances[accountLayout.liquidators[partyA][1]] += lf / 2;
            }
```
But in order to get into this array, a liquidator will need to call only one of the functions liquidatePartyA or setSymbolPrices.
Therefore there is no actual need to do the full process of liquidation in order to get the liquidation fee.

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L31

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L88

Given this scenario liquidators will be tempted to do only half of the liquidation process, as they will still receive the full liquidation fee at the end when someone else finishes the liquidation process.

## Impact
Malicious liquidators can receive the liquidation fee without finishing the full liquidation of Party A.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L20

https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/liquidation/LiquidationFacetImpl.sol#L34

## Tool used

Manual Review

## Recommendation
Given the issue describe above, it should be a better approach to give the liquidation fee at the msg.sender which liquidated all of the positions in the function liquidatePositionsPartyA. This way the liquidator will be more motivated to liquidate all of the positions and receive the whole liquidation reward than only doing half of the liquidation process.

# 7. Force closing a position won't work when the order type is MARKET.

## Summary
Force closing a position won't work when the order type is MARKET.

## Vulnerability Detail
Duo to the require statement applied, force closing a position is only possible if the order type is LIMIT, which shouldn't be like this.

```solidity
    function forceClosePosition(uint256 quoteId, PairUpnlAndPriceSig memory upnlSig) internal {
        AccountStorage.Layout storage accountLayout = AccountStorage.layout();
        MAStorage.Layout storage maLayout = MAStorage.layout();
        Quote storage quote = QuoteStorage.layout().quotes[quoteId];

        uint256 filledAmount = quote.quantityToClose;
        require(quote.quoteStatus == QuoteStatus.CLOSE_PENDING, "PartyAFacet: Invalid state");
        require(
            block.timestamp > quote.modifyTimestamp + maLayout.forceCloseCooldown,
            "PartyAFacet: Cooldown not reached"
        );
        require(block.timestamp <= quote.deadline, "PartyBFacet: Quote is expired");
        require(
            quote.orderType == OrderType.LIMIT,
            "PartyBFacet: Quote's order type should be LIMIT"
        );
        if (quote.positionType == PositionType.LONG) {
            require(
                upnlSig.price >=
                    quote.requestedClosePrice +
                        (quote.requestedClosePrice * maLayout.forceCloseGapRatio) /
                        1e18,
                "PartyAFacet: Requested close price not reached"
            );
        } else {
            require(
                upnlSig.price <=
                    quote.requestedClosePrice -
                        (quote.requestedClosePrice * maLayout.forceCloseGapRatio) /
                        1e18,
                "PartyAFacet: Requested close price not reached"
            );
        }

        LibMuon.verifyPairUpnlAndPrice(upnlSig, quote.partyB, quote.partyA, quote.symbolId);
        LibSolvency.isSolventAfterClosePosition(
            quoteId,
            filledAmount,
            quote.requestedClosePrice,
            upnlSig
        );
        accountLayout.partyANonces[quote.partyA] += 1;
        accountLayout.partyBNonces[quote.partyB][quote.partyA] += 1;
        LibQuote.closeQuote(quote, filledAmount, quote.requestedClosePrice);
    }
```
The difference between LIMIT and MARKET orders when closing a position. Is that the MARKET order requires that the whole quantity of the Quote is being closed, while the LIMIT can be partially closed.

```solidity
        if (quote.orderType == OrderType.LIMIT) {
            require(quote.quantityToClose >= filledAmount, "PartyBFacet: Invalid filledAmount");
        } else {
            require(quote.quantityToClose == filledAmount, "PartyBFacet: Invalid filledAmount");
        }
```
Generally there shouldn't be restrictions regarding MARKET orders when force closing a position, which means that this is a bug in the protocol design.

## Impact
Force closing a position won't work when the order type is MARKET.

## Code Snippet
https://github.com/sherlock-audit/2023-06-symmetrical/blob/main/symmio-core/contracts/facets/PartyA/PartyAFacetImpl.sol#L253

## Tool used

Manual Review

## Recommendation
Consider removing the require statement, as both LIMIT and MARKET orders should be permitted to be force closed as a position.

