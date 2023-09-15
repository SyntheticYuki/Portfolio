![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Funstoppable.jpg&w=256&q=75)

# [Unstoppable](https://audits.sherlock.xyz/contests/95)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Unstoppable | [Website](https://www.unstoppable.ooo) | [Twitter](https://twitter.com/unstoppablefi) | 24,000 USDC | 1265 | 7 days | June 28, 2023 | July 5, 2023 |


## What is Unstoppable?
Instead of building a DEX to compete with existing DEXs like Uniswap, Sushiswap, GMX, etc the Unstoppable:DEX builds on top of these proven liquidity sources to provide: 
- An advanced Spot trading experience, not seen on Uniswap/Sushiswap
- And a next-generation margin trading experience not possible on platforms like GMX.

### Spot
Unstoppable:DEX Spot provides all the advanced order types available on a CEX but rarely seen in DeFi.
- Buy & Sell Limit Orders
- Trailing Stop Loss
- DCA (Dollar-Cost-Average)
An exchange is only as good as the trades you can execute there. Traders must be able to trade in size without being exposed to excessive slippage or position constraints. Unstoppable:DEX enables this by aggregating existing deep liquidity sources.

### Margin
Unstoppable:DEX Margin provides unique solutions for margin trading by building on top of existing DeFi.

- liquidity and backing all our trades 1:1 with the underlying asset (borrowed from our LPs)
- providing safe and under-collateralized borrowing options for our traders.
  
When opening a trade instead of competing against LPs (like GMX) you are simply borrowing single-sided liquidity. This allows Unstoppable to back all leveraged trads 1:1 with spot assets. 
This design removes the need to keep Open Interest balanced like on other platforms, meaning there are no Open Interest caps or position size limits for our traders. 

## Audit scope
- [unstoppable-dex-audit/contracts/margin-dex.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/MarginDex.vy)
- [unstoppable-dex-audit/contracts/margin-dex/SwapRouter.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/SwapRouter.vy)
- [unstoppable-dex-audit/contracts/margin-dex/Vault.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy)
- [unstoppable-dex-audit/contracts/spot-dex/Dca.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/Dca.vy)
- [unstoppable-dex-audit/contracts/spot-dex/LimitOrders.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/LimitOrders.vy)
- [unstoppable-dex-audit/contracts/spot-dex/TrailingStopDex.vy](https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/spot-dex/TrailingStopDex.vy)

## Issues found by Yuki

| Severity | Title | Count |
|:--|:--|:--:|
| High | Wrong accounting of the storage balances results for the protocol to be in debt even when the bad debt is repaid. | H-01 |
| High | Stuck funds in the vault duo to wrong logic applied when adding margin to a position. | H-02 |
| Medium | Order's minimum amount out is calculated wrongly when partially closing a position. | M-01 |

# H-01. Wrong accounting of the storage balances results for the protocol to be in debt even when the bad debt is repaid.
## Summary
Wrong accounting of the storage balances results for the protocol to be in debt even when the bad debt is repaid.

## Vulnerability Detail
When a position is fully closed, it can result to accruing bad debt or being in profit and repaying the borrowed liquidity which all depends on the amount_out_received from the swap.

In a bad debt scenario the function calculates the bad debt and accrues it based on the difference between the position debt and the amount out received from the swap. And after that repays the liquidity providers with the same received amount from the swap.

```vy
    if amount_out_received >= position_debt_amount:
        # all good, LPs are paid back, remainder goes back to trader
        trader_pnl: uint256 = amount_out_received - position_debt_amount
        self.margin[position.account][position.debt_token] += trader_pnl
        self._repay(position.debt_token, position_debt_amount)
    else:
        # edge case: bad debt
        self.is_accepting_new_orders = False  # put protocol in defensive mode
        bad_debt: uint256 = position_debt_amount - amount_out_received
        self.bad_debt[position.debt_token] += bad_debt
        self._repay(
            position.debt_token, amount_out_received
        )  # repay LPs as much as possible
        log BadDebt(position.debt_token, bad_debt, position.uid)
```

The mapping total_debt_amount holds all debt borrowed from users, and this amount accrues interest with the time.

Prior to closing a position, the bad debt is calculated and accrued to the mapping bad_debt, but it isn't subtracted from the mapping total_debt_amount which holds all the debt even the bad one accrued through the time. 

```vy
@internal
def _update_debt(_debt_token: address):
    """
    @notice
        Accounts for any accrued interest since the last update.
    """
    if block.timestamp == self.last_debt_update[_debt_token]:
        return  # already up to date, nothing to do

    self.last_debt_update[_debt_token] = block.timestamp
    
    if self.total_debt_amount[_debt_token] == 0:
        return # no debt, no interest

    self.total_debt_amount[_debt_token] += self._debt_interest_since_last_update(
        _debt_token
    )
```

As the bad debt isn't subtracted from the total_debt_amount when closing a position, even after repaying the bad debt. It will still be in the total_debt_amount, which will prevent the full withdrawing of the liquidity funds.
```vy
@nonreentrant("lock")
@external
def repay_bad_debt(_token: address, _amount: uint256):
    """
    @notice
        Allows to repay bad_debt in case it was accrued.
    """
    self.bad_debt[_token] -= _amount
    self._safe_transfer_from(_token, msg.sender, self, _amount)
```
```vy
@internal
@view
def _available_liquidity(_token: address) -> uint256:
    return self._total_liquidity(_token) - self.total_debt_amount[_token]
```
```vy
@internal
@view
def _total_liquidity(_token: address) -> uint256:
    return (
        self.base_lp_total_amount[_token]
        + self.safety_module_lp_total_amount[_token]
        - self.bad_debt[_token]
    )
```

## Impact
The issue leads to liquidity providers unable to withdraw their full amount of funds, as even after repaying the bad debt it will still be in the total_debt_amount mapping.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L233

## Tool used

Manual Review

## Recommendation
The only way to fix this problem is to repay the position debt amount prior to closing a position and not only the received amount from the swap. Because total_debt_amount holds the bad debt as well.

```vy
    else:
        # edge case: bad debt
        self.is_accepting_new_orders = False  # put protocol in defensive mode
        bad_debt: uint256 = position_debt_amount - amount_out_received
        self.bad_debt[position.debt_token] += bad_debt
        self._repay(
            position.debt_token, position_debt_amount
        )  # repay LPs as much as possible
        log BadDebt(position.debt_token, bad_debt, position.uid)

```

# H-02. Stuck funds in the vault duo to wrong logic applied when adding margin to a position.
## Summary
Stuck funds in the vault duo to wrong logic applied when adding margin to a position. 

## Vulnerability Detail
The function add_margin is used by user to reduce the leverage of the position, by adding margin to it. 

By using the function, the position's margin amount is updated with the new amount added, so that the position's leverage doesn't exceed the max leverage allowed and the function isn't eligible for liquidation. 

Once added this margin can't be removed from the position if it gets the position back in liquidation state.

```vy
@external
def add_margin(_position_uid: bytes32, _amount: uint256):
    """
    @notice
        Allows to add additional margin to a Position and 
        reduce the leverage.
    """
    assert self.is_whitelisted_dex[msg.sender], "unauthorized"

    position: Position = self.positions[_position_uid]

    assert (self.margin[position.account][position.debt_token] >= _amount), "not enough margin"

    self.margin[position.account][position.debt_token] -= _amount
    position.margin_amount += _amount

    self.positions[_position_uid] = position
    log MarginAdded(_position_uid, _amount)
```
```vy
@external
def remove_margin(_position_uid: bytes32, _amount: uint256):
    """
    @notice
        Allows to remove margin from a Position and 
        increase the leverage.
    """
    assert self.is_whitelisted_dex[msg.sender], "unauthorized"

    position: Position = self.positions[_position_uid]

    assert position.margin_amount >= _amount, "not enough margin"

    position.margin_amount -= _amount
    self.margin[position.account][position.debt_token] += _amount

    assert not self._is_liquidatable(_position_uid), "exceeds max leverage"
    
    self.positions[_position_uid] = position
    log MarginRemoved(_position_uid, _amount)
```

The only choice would be to close or partially close the position.

But if we look over the function we can see that the margin amount isn't accounted anywhere.

Fully closing the position: Even tho margin was added the position_amount stays the same, so the function doesn't count it and still process the closing with the old values. At the end the extra margin added is being deleted with the position, which means that the funds are stuck in the contract.

Partially closing the position: On the other hand partially closing counts the margin amount and calculates the debt ratio based on the new amount of margin, but still do the closing based on the old leverage before the extra margin was added. Which leads to the same stuck funds in the contract as fully closing the position.

The conclusion is that adding margin to the position can reduce the leverage of the position, so the position is not eligible for liquidation, but closing the position is still based on the old leverage before the extra margin was added to the position. In the end when closing the positions, these extra funds added to the position are not included anywhere   and are stuck in the contract.

```vy
@internal
def _close_position(_position_uid: bytes32, _min_amount_out: uint256) -> uint256:
    """
    @notice
        Closes an existing position, repays the debt plus
        accrued interest and credits/debits the users margin
        with the remaining PnL.
    """
    # fetch the position from the positions-dict by uid
    position: Position = self.positions[_position_uid]

    # assign to local variable to make it editable
    min_amount_out: uint256 = _min_amount_out
    if min_amount_out == 0:
        # market order, add some slippage protection
        min_amount_out = self._market_order_min_amount_out(
            position.position_token, position.debt_token, position.position_amount
        )

    position_debt_amount: uint256 = self._debt(_position_uid)
    amount_out_received: uint256 = self._swap(
        position.position_token,
        position.debt_token,
        position.position_amount,
        min_amount_out,
    )

    if amount_out_received >= position_debt_amount:
        # all good, LPs are paid back, remainder goes back to trader
        trader_pnl: uint256 = amount_out_received - position_debt_amount
        self.margin[position.account][position.debt_token] += trader_pnl
        self._repay(position.debt_token, position_debt_amount)
    else:
        # edge case: bad debt
        self.is_accepting_new_orders = False  # put protocol in defensive mode
        bad_debt: uint256 = position_debt_amount - amount_out_received
        self.bad_debt[position.debt_token] += bad_debt
        self._repay(
            position.debt_token, amount_out_received
        )  # repay LPs as much as possible
        log BadDebt(position.debt_token, bad_debt, position.uid)

    # cleanup position
    self.positions[_position_uid] = empty(Position)

    log PositionClosed(position.account, position.uid, position, amount_out_received)

    return amount_out_received
```

```vy
@nonreentrant("lock")
@external
def reduce_position(
    _position_uid: bytes32, _reduce_by_amount: uint256, _min_amount_out: uint256
) -> uint256:
    """
    @notice
        Partially closes an existing position, by selling some of the 
        underlying position_token.
        Reduces both debt and margin in the position, leverage 
        remains as is.
    """
    assert self.is_whitelisted_dex[msg.sender], "unauthorized"
    assert not self._is_liquidatable(_position_uid), "in liquidation"

    position: Position = self.positions[_position_uid]
    assert position.position_amount >= _reduce_by_amount, "_reduce_by_amount > position"

    min_amount_out: uint256 = _min_amount_out
    if min_amount_out == 0:
        # market order, add some slippage protection
        min_amount_out = self._market_order_min_amount_out(
            position.position_token, position.debt_token, position.position_amount
        )

    debt_amount: uint256 = self._debt(_position_uid)
    margin_debt_ratio: uint256 = position.margin_amount * PRECISION / debt_amount

    amount_out_received: uint256 = self._swap(
        position.position_token, position.debt_token, _reduce_by_amount, min_amount_out
    )

    # reduce margin and debt, keep leverage as before
    reduce_margin_by_amount: uint256 = (
        amount_out_received * margin_debt_ratio / PRECISION
    )
    reduce_debt_by_amount: uint256 = amount_out_received - reduce_margin_by_amount

    position.margin_amount -= reduce_margin_by_amount

    burnt_debt_shares: uint256 = self._repay(position.debt_token, reduce_debt_by_amount)
    position.debt_shares -= burnt_debt_shares
    position.position_amount -= _reduce_by_amount

    self.positions[_position_uid] = position

    log PositionReduced(position.account, _position_uid, position, amount_out_received)

    return amount_out_received
```

## Impact
Duo to the wrong logic applied, this issue leads to permanent loss of funds as the funds are stuck in the vault.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L233

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L290

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L503

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L528

## Tool used

Manual Review

## Recommendation
Adding margin should not only update the position's margin but update whole position amount by doing a swap with the extra margin provided.

```vy
@external
def add_margin(_position_uid: bytes32, _amount: uint256, _min_position_amount_out: uint256):
    """
    @notice
        Allows to add additional margin to a Position and 
        reduce the leverage.
    """
    assert self.is_whitelisted_dex[msg.sender], "unauthorized"

    position: Position = self.positions[_position_uid]

    assert (self.margin[position.account][position.debt_token] >= _amount), "not enough margin"

    self.margin[position.account][position.debt_token] -= _amount

    amount_bought: uint256 = self._swap(
        _debt_token, _position_token, _amount, _min_position_amount_out
    )

    position.margin_amount += _amount
    position.position_amount += _amount

    fee: uint256 = _amount * self.trade_open_fee / PERCENTAGE_BASE
    assert self.margin[_account][_debt_token] >= fee, "not enough margin for fee"
    self.margin[_account][_debt_token] -= fee
    self._distribute_trading_fee(_debt_token, fee)

    self.positions[_position_uid] = position
    log MarginAdded(_position_uid, _amount)
```

# M-01. Order's minimum amount out is calculated wrongly when partially closing a position.
## Summary
Order's minimum amount out is calculated wrongly when partially closing a position.

## Vulnerability Detail
When a user partially closes a position, the user can either set the wanted _min_amount_out by himself or apply "_min_amount_out" as zero and let the function calculate the minimum amount out itself based on the position's info.

However the function makes the mistake to calculate the minimum amount out based on the whole position amount instead of calculating it based on the amount that the user wants to reduce the position for e.g - "_reduce_by_amount".

In the end the minimum amount out calculated by the function will be never met, as the user is partially closing an amount from the position and not closing the full amount of the position. Therefore this feature leads to permanent revert.

```vy
@nonreentrant("lock")
@external
def reduce_position(
    _position_uid: bytes32, _reduce_by_amount: uint256, _min_amount_out: uint256
) -> uint256:
    """
    @notice
        Partially closes an existing position, by selling some of the 
        underlying position_token.
        Reduces both debt and margin in the position, leverage 
        remains as is.
    """
    assert self.is_whitelisted_dex[msg.sender], "unauthorized"
    assert not self._is_liquidatable(_position_uid), "in liquidation"

    position: Position = self.positions[_position_uid]
    assert position.position_amount >= _reduce_by_amount, "_reduce_by_amount > position"

    min_amount_out: uint256 = _min_amount_out
    if min_amount_out == 0:
        # market order, add some slippage protection
        min_amount_out = self._market_order_min_amount_out(
            position.position_token, position.debt_token, position.position_amount
        )

    debt_amount: uint256 = self._debt(_position_uid)
    margin_debt_ratio: uint256 = position.margin_amount * PRECISION / debt_amount

    amount_out_received: uint256 = self._swap(
        position.position_token, position.debt_token, _reduce_by_amount, min_amount_out
    )

    # reduce margin and debt, keep leverage as before
    reduce_margin_by_amount: uint256 = (
        amount_out_received * margin_debt_ratio / PRECISION
    )
    reduce_debt_by_amount: uint256 = amount_out_received - reduce_margin_by_amount

    position.margin_amount -= reduce_margin_by_amount

    burnt_debt_shares: uint256 = self._repay(position.debt_token, reduce_debt_by_amount)
    position.debt_shares -= burnt_debt_shares
    position.position_amount -= _reduce_by_amount

    self.positions[_position_uid] = position

    log PositionReduced(position.account, _position_uid, position, amount_out_received)

    return amount_out_received
```

## Impact
Duo to the logic error, the minimum amount calculated by the function will be incorrect which leads to the permanent revert of this feature.

## Code Snippet

https://github.com/sherlock-audit/2023-06-unstoppable/blob/main/unstoppable-dex-audit/contracts/margin-dex/Vault.vy#L290

## Tool used

Manual Review

## Recommendation
As the function reduce_position is used to partially close a position, the min amount out should be calculated based on the user's _reduce_by_amount and not the whole position amount. As this problem leads to the incorrect calculation of the minimum amount out.

```vy
    min_amount_out: uint256 = _min_amount_out
    if min_amount_out == 0:
        # market order, add some slippage protection
        min_amount_out = self._market_order_min_amount_out(
            position.position_token, position.debt_token, _reduce_by_amount
        )

```


