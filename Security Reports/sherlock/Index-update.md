![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2FIndex.jpg&w=256&q=75)

# [Index-update](https://audits.sherlock.xyz/contests/91)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Index-update | [Website](https://indexcoop.com/) | [Twitter](https://twitter.com/indexcoop) | 24,000 USDC | 1418 | 7 days | July 10, 2023 | July 17, 2023 |

## What is Index?
Index Protocol is an EVM-based protocol that enables asset management strategies by translating them into ERC-20 tokens. It is a good-faith fork of Set Protocol v2 and will be the primary platform for Index Coop products beginning in 2023. In contrast to Set Protocol v2, Index Protocol is not currently intended to be a self-service platform for third parties to launch products permissionlessly; rather, it exists to support secure and accessible structured products launched by Index Coop and its partners.

The following Index Coop products have been deployed on Index Protocol:
- [Diversified Staked ETH Index (dsETH)](https://docs.indexcoop.com/index-coop-community-handbook/products/diversified-staked-eth-index-dseth)
- [Gitcoin ETH Index (gtcETH)](https://docs.indexcoop.com/index-coop-community-handbook/products/gitcoin-eth-index-gtceth)
- [Money Market Index (icSMMT)](https://docs.indexcoop.com/index-coop-community-handbook/products/money-market-index-icsmmt)

## Audit scope

- [contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol)
- [contracts/protocol/integration/auction-price/BoundedStepwiseLinearPriceAdapter.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseLinearPriceAdapter.sol)
- [contracts/protocol/integration/auction-price/BoundedStepwiseLogarithmicPriceAdapter.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseLogarithmicPriceAdapter.sol)
- [contracts/protocol/integration/auction-price/ConstantPriceAdapter.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/ConstantPriceAdapter.sol)
- [contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol)
- [contracts/protocol/modules/v1/BasicIssuanceModule.sol](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/BasicIssuanceModule.sol)

## Issues found by Yuki

| Severity | Title | Count |
|:--|:--|:--:|
| High | SetToken can't be unlocked early. | 1 |

# 1. SetToken can't be unlocked early.
## Summary
SetToken can't be unlocked early

## Vulnerability Detail
The function unlock() is used to unlock the setToken after rebalancing, as how it is right now there are two ways to unlock the setToken.

- can be unlocked once the rebalance duration has elapsed
- can be unlocked early if all targets are met, there is excess or at-target quote asset, and raiseTargetPercentage is zero
```solidity
    function unlock(ISetToken _setToken) external {
        bool isRebalanceDurationElapsed = _isRebalanceDurationElapsed(_setToken);
        bool canUnlockEarly = _canUnlockEarly(_setToken);

        // Ensure that either the rebalance duration has elapsed or the conditions for early unlock are met
        require(isRebalanceDurationElapsed || canUnlockEarly, "Cannot unlock early unless all targets are met and raiseTargetPercentage is zero");

        // If unlocking early, update the state
        if (canUnlockEarly) {
            delete rebalanceInfo[_setToken].rebalanceDuration;
            emit LockedRebalanceEndedEarly(_setToken);
        }

        // Unlock the SetToken
        _setToken.unlock();
    }
```
```solidity
    function _canUnlockEarly(ISetToken _setToken) internal view returns (bool) {
        RebalanceInfo storage rebalance = rebalanceInfo[_setToken];
        return _allTargetsMet(_setToken) && _isQuoteAssetExcessOrAtTarget(_setToken) && rebalance.raiseTargetPercentage == 0;
    }
```

The main problem occurs as the value of raiseTargetPercentage isn't reset after rebalancing. The other thing is that the function setRaiseTargetPercentage can't be used to fix this issue as it doesn't allow giving raiseTargetPercentage a zero value.

A setToken can use the AuctionModule to rebalance multiple times, duo to the fact that raiseTargetPercentage value isn't reset after every rebalancing. Once changed with the help of the function setRaiseTargetPercentage this value will only be non zero for every next rebalancing. A setToken can be unlocked early only if all other requirements are met and the raiseTargetPercentage equals zero.

This problem prevents for a setToken to be unlocked early on the next rebalances, once the value of the variable raiseTargetPercentage is set to non zero. 

On every rebalance a manager should be able to keep the value of raiseTargetPercentage to zero (so the setToken can be unlocked early), or increase it at any time with the function setRaiseTargetPercentage.

```solidity
    function setRaiseTargetPercentage(
        ISetToken _setToken,
        uint256 _raiseTargetPercentage
    )
        external
        onlyManagerAndValidSet(_setToken)
    {
        // Ensure the raise target percentage is greater than 0
        require(_raiseTargetPercentage > 0, "Target percentage must be greater than 0");

        // Update the raise target percentage in the RebalanceInfo struct
        rebalanceInfo[_setToken].raiseTargetPercentage = _raiseTargetPercentage;

        // Emit an event to log the updated raise target percentage
        emit RaiseTargetPercentageUpdated(_setToken, _raiseTargetPercentage);
    }
```

## Impact
Once the value of raiseTargetPercentage is set to non zero, every next rebalancing of the setToken won't be eligible for unlocking early. As the value of raiseTargetPercentage isn't reset after every rebalance and neither the manager can set it back to zero with the function setRaiseTargetPercentage().

## Code Snippet

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L389

## Tool used

Manual Review

## Recommendation
Recommend to reset the value raiseTargetPercentage after every rebalancing.

```solidity
    function unlock(ISetToken _setToken) external {
        bool isRebalanceDurationElapsed = _isRebalanceDurationElapsed(_setToken);
        bool canUnlockEarly = _canUnlockEarly(_setToken);

        // Ensure that either the rebalance duration has elapsed or the conditions for early unlock are met
        require(isRebalanceDurationElapsed || canUnlockEarly, "Cannot unlock early unless all targets are met and raiseTargetPercentage is zero");

        // If unlocking early, update the state
        if (canUnlockEarly) {
            delete rebalanceInfo[_setToken].rebalanceDuration;
            emit LockedRebalanceEndedEarly(_setToken);
        }

+       rebalanceInfo[_setToken].raiseTargetPercentage = 0;

        // Unlock the SetToken
        _setToken.unlock();
    }
```
