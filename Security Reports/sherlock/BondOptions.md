![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2FBond%20Protocol.jpg&w=256&q=75)

# [BondOptions](https://audits.sherlock.xyz/contests/99)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| BondOptions | [Website](https://bondprotocol.finance/) | [Twitter](https://twitter.com/Bond_Protocol) | 16,000 USDC | 874 | 5 days | July 3, 2023 | July 8, 2023 |

## What is BondOptions?

Bond Protocol's Option System is a flexible system for unlocking the power of Option Token Liquidity Mining (OTLM) for projects of all sizes. They incorporated insights gained from bonding, notably designing a system that can be used with or without a price oracle.

Overview:
- Bond Protocol is a system to create OTC markets for any ERC20 token pair with optional vesting of the payout. The markets do not require maintenance and will manage bond prices per the methodology defined in the specific auctioneer contract. Bond issuers create markets that pay out a Payout Token in exchange for deposited Quote Tokens. If payouts are instant, users can purchase Payout Tokens with Quote Tokens at the current market price and receive the Payout tokens immediately on purchase. Otherwise, they receive Bond Tokens to represent their position while their bond vests. Once the Bond Tokens vest, they can redeem it for the Quote Tokens.

## Audit scope

- [options/src/bases/OptionToken.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/bases/OptionToken.sol)
- [options/src/fixed-strike/FixedStrikeOptionTeller.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol)
- [options/src/fixed-strike/FixedStrikeOptionToken.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionToken.sol)
- [options/src/fixed-strike/liquidity-mining/OTLM.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol)
- [options/src/fixed-strike/liquidity-mining/OTLMFactory.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLMFactory.sol)
- [options/src/periphery/TokenAllowlist.sol](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/periphery/TokenAllowlist.sol)

## Issues found by Yuki

| Severity | Title | Count |
|:--|:--|:--:|
| High | Malicious user is able to drain the FixedStrikeOptionTeller contract. | 1 |
| High | Last epoch claimed for the user isn't updated on staking, which can permanently stuck the user from claiming rewards. | 2 |
| Medium | The function claimNextEpochRewards can't be used to move through epochs one txn at a time as it doesn't update the mapping lastEpochClaimed. | 3 |

# 1. Malicious user is able to drain the FixedStrikeOptionTeller contract.

## Summary
Malicious user is able to drain the FixedStrikeOptionTeller contract.

## Vulnerability Detail
Users are free to deploy their own option tokens and use the functions FixedStrikeOptionTeller.
The contract holds all the token balances from all of the option tokens created.

The problem here occurs, when the receiver reclaims collateral from expired option tokens. The withdrawn tokens are based on the option token total supply which is correct, but the receiver can call the function more than once as the total supply of the option token stays the same.

1. Malicious user sees which tokens the FixedStrikeOptionTeller posses.
2. Malicious user deploys a new option token based on which token he wants to steal and himself as the receiver.
3. Once deployed the malicious user calls the function create and provides tokens to the contract.
4. Malicious user waits for the option token to expire and calls the function reclaim and claims tokens till he drains the whole contract.

```solidity
    function deploy(
        ERC20 payoutToken_,
        ERC20 quoteToken_,
        uint48 eligible_,
        uint48 expiry_,
        address receiver_,
        bool call_,
        uint256 strikePrice_
    ) external override nonReentrant returns (FixedStrikeOptionToken) {
        // If eligible is zero, use current timestamp
        if (eligible_ == 0) eligible_ = uint48(block.timestamp);

        // Eligible and Expiry are rounded to the nearest day at 0000 UTC (in seconds) since
        // option tokens are only unique to a day, not a specific timestamp.
        eligible_ = uint48(eligible_ / 1 days) * 1 days;
        expiry_ = uint48(expiry_ / 1 days) * 1 days;

        // Revert if eligible is in the past, we do this to avoid duplicates tokens with the same parameters otherwise
        // Truncate block.timestamp to the nearest day for comparison
        if (eligible_ < uint48(block.timestamp / 1 days) * 1 days)
            revert Teller_InvalidParams(2, abi.encodePacked(eligible_));

        // Revert if the difference between eligible and expiry is less than min duration or eligible is after expiry
        // Don't need to check expiry against current timestamp since eligible is already checked
        if (eligible_ > expiry_ || expiry_ - eligible_ < minOptionDuration)
            revert Teller_InvalidParams(3, abi.encodePacked(expiry_));

        // Revert if any addresses are zero or the tokens are not contracts
        if (address(payoutToken_) == address(0) || address(payoutToken_).code.length == 0)
            revert Teller_InvalidParams(0, abi.encodePacked(payoutToken_));
        if (address(quoteToken_) == address(0) || address(quoteToken_).code.length == 0)
            revert Teller_InvalidParams(1, abi.encodePacked(quoteToken_));
        if (receiver_ == address(0)) revert Teller_InvalidParams(4, abi.encodePacked(receiver_));

        // Revert if strike price is zero or out of bounds
        int8 priceDecimals = _getPriceDecimals(strikePrice_, quoteToken_.decimals()); // @audit determine if this external call to provided quote token is an issue
        if (strikePrice_ == 0 || priceDecimals > int8(9) || priceDecimals < int8(-9))
            revert Teller_InvalidParams(6, abi.encodePacked(strikePrice_));

        // Create option token if one doesn't already exist
        // Timestamps are truncated above to give canonical version of hash
        bytes32 optionHash = _getOptionTokenHash(
            payoutToken_,
            quoteToken_,
            eligible_,
            expiry_,
            receiver_,
            call_,
            strikePrice_
        );

        FixedStrikeOptionToken optionToken = optionTokens[optionHash];

        // If option token doesn't exist, deploy it
        if (address(optionToken) == address(0)) {
            optionToken = _deploy(
                payoutToken_,
                quoteToken_,
                eligible_,
                expiry_,
                receiver_,
                call_,
                strikePrice_
            );

            // Set the domain separator for the option token on creation to save gas on permit approvals
            optionToken.updateDomainSeparator();

            // Store option token against computed hash
            optionTokens[optionHash] = optionToken;
        }
        return optionToken;
    }
```

```solidity
    function create(
        FixedStrikeOptionToken optionToken_,
        uint256 amount_
    ) external override nonReentrant {
        // Load option parameters
        (
            ERC20 payoutToken,
            ERC20 quoteToken,
            uint48 eligible,
            uint48 expiry,
            address receiver,
            bool call,
            uint256 strikePrice
        ) = optionToken_.getOptionParameters();

        // Retrieve the internally stored option token with this configuration
        // Reverts internally if token doesn't exist
        FixedStrikeOptionToken optionToken = getOptionToken(
            payoutToken,
            quoteToken,
            eligible,
            expiry,
            receiver,
            call,
            strikePrice
        );

        // Revert if provided token address does not match stored token address
        if (optionToken_ != optionToken) revert Teller_UnsupportedToken(address(optionToken_));

        // Revert if expiry is in the past
        if (uint256(expiry) < block.timestamp) revert Teller_OptionExpired(expiry);

        // Transfer in collateral
        // If call option, transfer in payout tokens equivalent to the amount of option tokens being issued
        // If put option, transfer in quote tokens equivalent to the amount of option tokens being issued * strike price
        if (call) {
            // Transfer payout tokens from user
            // Check that amount received is not less than amount expected
            // Handles edge cases like fee-on-transfer tokens (which are not supported)
            uint256 startBalance = payoutToken.balanceOf(address(this));
            payoutToken.safeTransferFrom(msg.sender, address(this), amount_);
            uint256 endBalance = payoutToken.balanceOf(address(this));
            if (endBalance < startBalance + amount_)
                revert Teller_UnsupportedToken(address(payoutToken));
        } else {
            // Calculate amount of quote tokens required to mint
            uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals());

            // Transfer quote tokens from user
            // Check that amount received is not less than amount expected
            // Handles edge cases like fee-on-transfer tokens (which are not supported)
            uint256 startBalance = quoteToken.balanceOf(address(this));
            quoteToken.safeTransferFrom(msg.sender, address(this), quoteAmount);
            uint256 endBalance = quoteToken.balanceOf(address(this));
            if (endBalance < startBalance + quoteAmount)
                revert Teller_UnsupportedToken(address(quoteToken));
        }

        // Mint new option tokens to sender
        optionToken.mint(msg.sender, amount_);
    }
```
```solidity
    function reclaim(FixedStrikeOptionToken optionToken_) external override nonReentrant {
        // Load option parameters
        (
            ERC20 payoutToken,
            ERC20 quoteToken,
            uint48 eligible,
            uint48 expiry,
            address receiver,
            bool call,
            uint256 strikePrice
        ) = optionToken_.getOptionParameters();

        // Retrieve the internally stored option token with this configuration
        // Reverts internally if token doesn't exist
        FixedStrikeOptionToken optionToken = getOptionToken(
            payoutToken,
            quoteToken,
            eligible,
            expiry,
            receiver,
            call,
            strikePrice
        );

        // Revert if token does not match stored token
        if (optionToken_ != optionToken) revert Teller_UnsupportedToken(address(optionToken_));

        // Revert if not expired
        if (uint48(block.timestamp) < expiry) revert Teller_NotExpired(expiry);

        // Revert if caller is not receiver
        if (msg.sender != receiver) revert Teller_NotAuthorized();

        // Transfer remaining collateral to receiver
        uint256 amount = optionToken.totalSupply();
        if (call) {
            payoutToken.safeTransfer(receiver, amount);
        } else {
            // Calculate amount of quote tokens equivalent to amount at strike price
            uint256 quoteAmount = amount.mulDiv(strikePrice, 10 ** payoutToken.decimals());
            quoteToken.safeTransfer(receiver, quoteAmount);
        }
    }
```

## Impact
Duo to receiver being able to reclaim his tokens more than once, a malicious user is able to exploit this issue by deploying an option token and himself as the receiver and drain the pool when reclaiming after the option token expires.

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L107

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L236

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395

## Tool used

Manual Review

## Recommendation
The best solution would be to make a guard for the function reclaim to revert once the receiver reclaimed his tokens.

```solidity
    function reclaim(FixedStrikeOptionToken optionToken_) external override nonReentrant {
        // Load option parameters
        (
            ERC20 payoutToken,
            ERC20 quoteToken,
            uint48 eligible,
            uint48 expiry,
            address receiver,
            bool call,
            uint256 strikePrice
        ) = optionToken_.getOptionParameters();

        // Retrieve the internally stored option token with this configuration
        // Reverts internally if token doesn't exist
        FixedStrikeOptionToken optionToken = getOptionToken(
            payoutToken,
            quoteToken,
            eligible,
            expiry,
            receiver,
            call,
            strikePrice
        );

        // Revert if token does not match stored token
        if (optionToken_ != optionToken) revert Teller_UnsupportedToken(address(optionToken_));

        if (reclaimed[optionToken] == true, "Revert");

        // Revert if not expired
        if (uint48(block.timestamp) < expiry) revert Teller_NotExpired(expiry);

        // Revert if caller is not receiver
        if (msg.sender != receiver) revert Teller_NotAuthorized();

        // Transfer remaining collateral to receiver
        uint256 amount = optionToken.totalSupply();
        reclaimed[optionToken] = true;

        if (call) {
            payoutToken.safeTransfer(receiver, amount);
        } else {
            // Calculate amount of quote tokens equivalent to amount at strike price
            uint256 quoteAmount = amount.mulDiv(strikePrice, 10 ** payoutToken.decimals());
            quoteToken.safeTransfer(receiver, quoteAmount);
        }
 
    }

```

# 2. Last epoch claimed for the user isn't updated on staking, which can permanently stuck the user from claiming rewards.

## Summary
Last epoch claimed for the user isn't updated on staking, which can permanently stuck the user from claiming rewards.

## Vulnerability Detail
When a user stakes for the first time, the function stake updated the user's rewards per token claimed to the current rewards per token stored, but forgets to update the user's last epoch claimed to the current epoch.

```solidity
    function stake(
        uint256 amount_,
        bytes calldata proof_
    ) external nonReentrant requireInitialized updateRewards tryNewEpoch {
        // Revert if deposits are disabled
        if (!depositsEnabled) revert OTLM_DepositsDisabled();

        // If allowlist configured, check if user is allowed to stake
        if (address(allowlist) != address(0)) {
            if (!allowlist.isAllowed(msg.sender, proof_)) revert OTLM_NotAllowed();
        }

        // Revert if deposit amount is zero to avoid zero transfers
        if (amount_ == 0) revert OTLM_InvalidAmount();

        // Get user balance, if non-zero, claim rewards before increasing stake
        uint256 userBalance = stakeBalance[msg.sender];
        if (userBalance > 0) {
            // Claim outstanding rewards, this will update the rewards per token claimed
            _claimRewards();
        } else {
            // Initialize the rewards per token claimed for the user to the stored rewards per token
            rewardsPerTokenClaimed[msg.sender] = rewardsPerTokenStored;
        }

        // Increase the user's stake balance and the total balance
        stakeBalance[msg.sender] = userBalance + amount_;
        totalBalance += amount_;

        // Transfer the staked tokens from the user to this contract
        stakedToken.safeTransferFrom(msg.sender, address(this), amount_);
    }
```

When claiming rewards for the user, the function will try to go over all of the past epochs to the current one. This all happens in a for loop, which calls the function _claimEpochRewards as much as the number of the past epochs. 

This could possible result to the transaction running out of gas, or needing an enormous amount of gas to proceed all of the calls made to the function _claimRewards so that the user can finally claim his rewards on the current epoch.

When deployed the OTML pool is intended to run forever and stakers are free to stake at any given time, so there could be a lot of passed epochs since the deployment of the pool. Which is problematic for later stakers, as they won't be able to claim their rewards and will simply emergency unstake. 

Note:
The function claimNextEpochRewards() can't be used to help the situation here. As on staking the user's rewardsPerTokenClaimed was updated to the current rewardsPerTokenStored, the function _claimEpochRewards will stop at the if statement (userRewardsClaimed >= rewardsPerTokenEnd) and return 0 before updating the mapping lastEpochClaimed to the next epoch in line. (For more info look at my other finding for claimNextEpochRewards).

```solidity
    function _claimRewards() internal returns (uint256) {
        // Claims all outstanding rewards for the user across epochs
        // If there are unclaimed rewards from epochs where the option token has expired, the rewards are lost

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

        // If the last epoch claimed is equal to the current epoch, then only try to claim for the current epoch
        if (userLastEpoch == epoch) return _claimEpochRewards(epoch);

        // If not, then the user has not claimed all rewards
        // Start at the last claimed epoch because they may not have completely claimed that epoch
        uint256 totalRewardsClaimed;
        for (uint48 i = userLastEpoch; i <= epoch; i++) {
            // For each epoch that the user has not claimed rewards for, claim the rewards
            totalRewardsClaimed += _claimEpochRewards(i);
        }

        return totalRewardsClaimed;
    }
```

```solidity
    function _claimEpochRewards(uint48 epoch_) internal returns (uint256) {
        // Claims all outstanding rewards for the user for the specified epoch
        // If the option token for the epoch has expired, the rewards are lost

        // Check that the epoch is valid
        if (epoch_ > epoch) revert OTLM_InvalidEpoch();

        // Get the rewards per token claimed by the user
        uint256 userRewardsClaimed = rewardsPerTokenClaimed[msg.sender];

        // Get the rewards per token at the start of the epoch and the rewards per token at the end of the epoch (start of the next one)
        // If the epoch is the current epoch, the rewards per token at the end of the epoch is the current rewards per token stored
        uint256 rewardsPerTokenStart = epochRewardsPerTokenStart[epoch_];
        uint256 rewardsPerTokenEnd = epoch_ == epoch
            ? rewardsPerTokenStored
            : epochRewardsPerTokenStart[epoch_ + 1];

        // If the user hasn't claimed the rewards up to the start of this epoch, then they have a previous unclaimed epoch
        // External functions protect against this by their ordering, but this makes it explicit
        if (userRewardsClaimed < rewardsPerTokenStart) revert OTLM_PreviousUnclaimedEpoch();

        // If user rewards claimed is greater than or equal to the rewards per token at the end of the epoch, then the user has already claimed all rewards for the epoch
        if (userRewardsClaimed >= rewardsPerTokenEnd) return 0;
```

## Impact
As last epoch claimed isn't updated to the current one on staking, later stakers could experience unable to claim their rewards duo to the function claim rewards running out of gas.

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L304

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L408

## Tool used

Manual Review

## Recommendation
The solution here is simple, update the lastEpochClaimed to the current epoch when the user stakes for the first time.

```solidity
    function stake(
        uint256 amount_,
        bytes calldata proof_
    ) external nonReentrant requireInitialized updateRewards tryNewEpoch {
        // Revert if deposits are disabled
        if (!depositsEnabled) revert OTLM_DepositsDisabled();

        // If allowlist configured, check if user is allowed to stake
        if (address(allowlist) != address(0)) {
            if (!allowlist.isAllowed(msg.sender, proof_)) revert OTLM_NotAllowed();
        }

        // Revert if deposit amount is zero to avoid zero transfers
        if (amount_ == 0) revert OTLM_InvalidAmount();

        // Get user balance, if non-zero, claim rewards before increasing stake
        uint256 userBalance = stakeBalance[msg.sender];
        if (userBalance > 0) {
            // Claim outstanding rewards, this will update the rewards per token claimed
            _claimRewards();
        } else {
            // Initialize the rewards per token claimed for the user to the stored rewards per token
            rewardsPerTokenClaimed[msg.sender] = rewardsPerTokenStored;
+           lastEpochClaimed[msg.sender] = epoch;
        }

        // Increase the user's stake balance and the total balance
        stakeBalance[msg.sender] = userBalance + amount_;
        totalBalance += amount_;

        // Transfer the staked tokens from the user to this contract
        stakedToken.safeTransferFrom(msg.sender, address(this), amount_);
    }
```

# 3. The function claimNextEpochRewards can't be used to move through epochs one txn at a time as it doesn't update the mapping lastEpochClaimed.

## Summary
The function claimNextEpochRewards can't be used to move through epochs one txn at a time as it doesn't update the mapping lastEpochClaimed.

## Vulnerability Detail
The function claimNextEpochRewards() is supposed to claim all outstanding rewards for the user on their next unclaimed epoch. And to allow moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

However this function doesn't work as intended and will be useless as it won't update the lastEpochClaimed to the next epoch in the line.

Imagine the following scenario:

1. User unstakes all of his funds and claims his rewards at epoch 10, his lastEpochClaimed will equal 10.
2. The user returns and stakes again at epoch 20, after the stake his rewardsPerTokenClaimed will equal the current rewardsPerTokenStored.
3. He calls the function claimNextEpochRewards in order to avoid gas issues.
4. In the function the else statement will be triggered as userClaimedRewardsPerToken > rewardsPerTokenEnd, so the function will try to claim rewards for the next epoch in line.
5. The problem is that the function _claimEpochRewards will return 0 before updating the user's lastEpochClaimed to the next epoch in line, as the if statement (userRewardsClaimed >= rewardsPerTokenEnd) will be triggered.

After the execution of the function claimNextEpochRewards, nothing changed and the lastEpochClaimed wasn't updated, which means that the user can't use the function to move through the epochs to avoid gas issues.

```solidity
    /// @notice Claim all outstanding rewards for the user for the next unclaimed epoch (and any remaining rewards from the previously claimed epoch)
    function claimNextEpochRewards()
        external
        nonReentrant
        updateRewards
        tryNewEpoch
        returns (uint256)
    {
        // Claims all outstanding rewards for the user on their next unclaimed epoch. Allows moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

        // Revert if user has no stake
        if (stakeBalance[msg.sender] == 0) revert OTLM_ZeroBalance();

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

        // If the last epoch claimed is equal to the current epoch, then try to claim for the current epoch
        if (userLastEpoch == epoch) return _claimEpochRewards(epoch);

        // If not, then the user has not claimed rewards from the next epoch
        // Check if the user has claimed all rewards from the last epoch first
        uint256 userClaimedRewardsPerToken = rewardsPerTokenClaimed[msg.sender];
        uint256 rewardsPerTokenEnd = epochRewardsPerTokenStart[userLastEpoch + 1];
        if (userClaimedRewardsPerToken < rewardsPerTokenEnd) {
            // If not, then claim the remaining rewards from the last epoch
            uint256 remainingLastEpochRewards = _claimEpochRewards(userLastEpoch);
            uint256 nextEpochRewards = _claimEpochRewards(userLastEpoch + 1);
            return remainingLastEpochRewards + nextEpochRewards;
        } else {
            // If so, then claim the rewards from the next epoch
            return _claimEpochRewards(userLastEpoch + 1);
        }
    }
```

## Impact
Duo to logic error, the function claimNextEpochRewards will be unusable when wanting to move through epochs one txn at a time, as in some scenarios it doesn't update the mapping lastEpochClaimed. 

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L430

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L485

## Tool used

Manual Review

## Recommendation
The solution for this problem is simple, you can update the lastEpochClaimed to the next epoch in line, if the function _claimEpochRewards returns 0.

The function _claimEpochRewards returning zero means that either of the if statements were trigger:

- if (userRewardsClaimed >= rewardsPerTokenEnd) return 0;
- if (uint256(optionToken.expiry()) < block.timestamp) return 0;

If option was expired the lastEpochClaimed would already be updated, but it doesn't if its updated twice since the value is the same.

```solidity
    function claimNextEpochRewards()
        external
        nonReentrant
        updateRewards
        tryNewEpoch
        returns (uint256)
    {
        // Claims all outstanding rewards for the user on their next unclaimed epoch. Allows moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

        // Revert if user has no stake
        if (stakeBalance[msg.sender] == 0) revert OTLM_ZeroBalance();

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

        // If the last epoch claimed is equal to the current epoch, then try to claim for the current epoch
        if (userLastEpoch == epoch) return _claimEpochRewards(epoch);

        // If not, then the user has not claimed rewards from the next epoch
        // Check if the user has claimed all rewards from the last epoch first
        uint256 userClaimedRewardsPerToken = rewardsPerTokenClaimed[msg.sender];
        uint256 rewardsPerTokenEnd = epochRewardsPerTokenStart[userLastEpoch + 1];
        if (userClaimedRewardsPerToken < rewardsPerTokenEnd) {
            // If not, then claim the remaining rewards from the last epoch
            uint256 remainingLastEpochRewards = _claimEpochRewards(userLastEpoch);
            uint256 nextEpochRewards = _claimEpochRewards(userLastEpoch + 1);
            return remainingLastEpochRewards + nextEpochRewards;
        } else {
            // If so, then claim the rewards from the next epoch
            uint256 rewards = _claimEpochRewards(userLastEpoch + 1);
            if (rewards == 0) lastEpochClaimed[msg.sender] = userLastEpoch + 1;
            return rewards;
        }
    }
```
