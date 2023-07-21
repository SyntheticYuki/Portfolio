![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Ftokensoft.jpg&w=256&q=75)

# [TokenSoft](https://audits.sherlock.xyz/contests/100)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| TokenSoft | [Website](https://www.tokensoft.io/) | [Twitter](https://twitter.com/TokensoftInc) | 14,000 USDC | 650 | 4 days | July 17, 2023 | July 21, 2023 |

## What is TokenSoft?

Tokensoft stands as a prominent provider of enterprise services and tools for blockchain foundations and projects. Our ultimate goal is to restore integrity to financial markets by seamlessly merging existing regulations with decentralized technology.
In the dynamic realm of digital assets, security, efficiency, and compliance take center stage. Tokensoft equips teams with robust digital asset solutions that empower them to confidently navigate their challenges with convenience.

## Audit scope

- [contracts/claim/BasicDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/BasicDistributor.sol)
- [contracts/claim/CrosschainContinuousV.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/CrosschainContinuousVestingMerkle.sol)
- [contracts/claim/CrosschainTrancheVesti.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/CrosschainTrancheVestingMerkle.sol)
- [contracts/claim/Satellite.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/Satellite.sol)
- [contracts/claim/abstract/AdvancedDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/AdvancedDistributor.sol)
- [contracts/claim/abstract/CrosschainDist.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/CrosschainDistributor.sol)
- [contracts/claim/abstract/CrosschainMerkleDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/CrosschainMerkleDistributor.sol)
- [contracts/claim/abstract/Distributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/Distributor.sol)
- [contracts/claim/abstract/MerkleSet.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/MerkleSet.sol)
- [contracts/claim/ContinuousVestingMerkle.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/ContinuousVestingMerkle.sol)
- [contracts/claim/PriceTierVestingMerkle.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingMerkle.sol)
- [contracts/claim/PriceTierVestingSale_2_0.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingSale_2_0.sol)
- [contracts/claim/abstract/TrancheVesting.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/TrancheVesting.sol)
- [contracts/utilities/Sweepable.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/utilities/Sweepable.sol)

## Issues found by Yuki

| Severity | Title | Count |
|:--|:--|:--:|
| High | Slightly increasing the vote factor can result to beneficiaries not able to claim their tokens. | 1 |
| High | User can initialize distribution record multiple times to extend his voting power. | 2 | 
| High | Logic error occurs when executing a claim, if the beneficiary was adjusted before. | 3 |
| High | Re-initializing the beneficiary if the total has been updated doesn't mint any additional voting power to the beneficiary. | 4 |
| High | Wrong accounting of voting power minted to the beneficiary in most of the vesting contracts. | 5 |

# 1. Slightly increasing the vote factor can result to beneficiaries not able to claim their tokens.

## Summary
Slightly increasing the vote factor can result to beneficiaries not able to claim their tokens.

## Vulnerability Detail
Increasing the vote factor can result to unexpected outcome, as beneficiaries won't be able to claim their tokens.

When executing claim, the function calculates the amount of votes it has to burn based on the formula in the function 
tokensToVotes. This can be problematic if the vote power increases and can lead to the following scenario:
- A beneficiary is initialized and voting power is minted to it based on the current factor.
- Time passes and the vote factor slightly increases
- The beneficiary tries to claim his tokens but as the vote factor increased, the function will try to burn more voting power than the user has and it will revert.
- In the end the beneficiary won't be able to claim his rewards.

```solidity
  function _executeClaim(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount);

    // reduce voting power through ERC20Votes extension
    _burn(beneficiary, tokensToVotes(_claimed));
  }
``` 
```solidity
  function tokensToVotes(uint256 tokenAmount) private view returns (uint256) {
    return (tokenAmount * voteFactor) / fractionDenominator;
  }
```

## Impact
Duo to difference between the two vote factors, when the beneficiary was initialized and after the vote factor increases. The beneficiary won't be able to claim his tokens, as the function will try to burn more voting power than the beneficiary has. 

## Code Snippet

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/AdvancedDistributor.sol#L87

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/AdvancedDistributor.sol#L73

## Tool used

Manual Review

## Recommendation
The one way to fix this issue would be to burn the whole amount of voting power the beneficiary has, only if the tokens claimed have more voting power than the user has. This will prevent the issue from not being able to claim rewards, if vote factor increases over time.

# 2. User can initialize distribution record multiple times to extend his voting power.
## Summary
User can initialize distribution record multiple times to extend his voting power.

## Vulnerability Detail
The function initializeDistributionRecord can be called by anyone in order to set up the distribution record for the given beneficiary.

Looking at the function, there is no particular check to see if the beneficiary was initialized before, which means that the function can be called multiple times.
- In general this is the right thing to do, as no matter how many times the function is called the distribution record won't change if the purchased amount is the same.
- However even tho the distribution record doesn't change, the function will mint voting power for the corresponding purchased amount every time it's called. 

Duo to this issue beneficiary is able to increase his voting power, by initializing his distribution record multiple times.

```solidity
  function initializeDistributionRecord(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) {
    _initializeDistributionRecord(beneficiary, getPurchasedAmount(beneficiary));
  }
```
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

    // add voting power through ERC20Votes extension
    _mint(beneficiary, tokensToVotes(totalAmount));
  }
```
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount);

    // Checks
    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }
```

## Impact
A beneficiary is able to increase his voting power, as even tho the distribution record doesn't change if it was previously initialized with the same purchased amount. The function still mints the voting power for the purchased amount, so every time the function is called, it will still increase the voting power of the beneficiary.

## Code Snippet

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingSale_2_0.sol#L91

## Tool used

Manual Review

## Recommendation
Recommend to not allow a beneficiary to be initialized multiple times with the same purchased amount. As an example a mapping can be made to store the last initialized value of the benificiary.

```solidity
  function initializeDistributionRecord(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) {
+   uint256 purchasedAmount = getPurchasedAmount(beneficiary);

+   require(lastInitialized[beneficiary] != purchasedAmount, "")

    _initializeDistributionRecord(beneficiary, purchasedAmount);
   
+   lastInitialized[beneficiary] = purchasedAmount;
  }
```

# 3. Logic error occurs when executing a claim, if the beneficiary was adjusted before.
## Summary
Logic error occurs when executing a claim, if the beneficiary was adjusted before.

## Vulnerability Detail
The owner is able to adjust the quantity claimable by a user, overriding the value in the distribution record by calling the function adjust.

```solidity
  function adjust(address beneficiary, int256 amount) external onlyOwner {
    DistributionRecord memory distributionRecord = records[beneficiary];
    require(distributionRecord.initialized, 'must initialize before adjusting');

    uint256 diff = uint256(amount > 0 ? amount : -amount);
    require(diff < type(uint120).max, 'adjustment > max uint120');

    if (amount < 0) {
      // decreasing claimable tokens
      require(total >= diff, 'decrease greater than distributor total');
      require(distributionRecord.total >= diff, 'decrease greater than distributionRecord total');
      total -= diff;
      records[beneficiary].total -= uint120(diff);
      token.safeTransfer(owner(), diff);
      // reduce voting power
      _burn(beneficiary, tokensToVotes(diff));
    } else {
      // increasing claimable tokens
      total += diff;
      records[beneficiary].total += uint120(diff);
      // increase voting pwoer
      _mint(beneficiary, tokensToVotes(diff));
    }

    emit Adjust(beneficiary, amount);
  }
```
However this adjusting won't work on most of the vesting contracts and can result to the wrong accounting of the Distribution records.

Takes for example the PriceTierVestingSale_2_0, where the total depends on the purchased amount by the beneficiary. Which can lead to the following scenario:
1. Owner adjusts a beneficiary - either decreasing or increasing his claimable tokens.
2. Beneficiary calls the claim function.
3. The distribution record was changed by the owner, but the function will still re-initialize the beneficiary to his purchased amount. Which means that the beneficiary will return to his old values before the adjustment by the owner.

```solidity
  function claim(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) nonReentrant {
    uint256 claimableAmount = getClaimableAmount(beneficiary);
    uint256 purchasedAmount = getPurchasedAmount(beneficiary);

    // effects
    uint256 claimedAmount = super._executeClaim(beneficiary, purchasedAmount);

    // interactions
    super._settleClaim(beneficiary, claimedAmount);
  }
```
```solidity
  function _executeClaim(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount);

    // reduce voting power through ERC20Votes extension
    _burn(beneficiary, tokensToVotes(_claimed));
  }
```
```solidity
  function _executeClaim(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);

    // effects
    if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount);

    // Checks
    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }
```

## Impact
Wrong accounting of the distribution record occurs, when a claim is executed after adjustment by the owner. As the claim function will re-initialize the beneficiary to his previous distribution record before the adjustment.

## Code Snippet

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/AdvancedDistributor.sol#L105

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingSale_2_0.sol#L97

## Tool used

Manual Review

## Recommendation
The system should track if the beneficiary was initialized before, and add/sub the difference when re-initializing.

```solidity
  function adjust(address beneficiary, int256 amount) external onlyOwner {
    DistributionRecord memory distributionRecord = records[beneficiary];
    require(distributionRecord.initialized, 'must initialize before adjusting');

    uint256 diff = uint256(amount > 0 ? amount : -amount);
    require(diff < type(uint120).max, 'adjustment > max uint120');

    if (amount < 0) {
      // decreasing claimable tokens
      require(total >= diff, 'decrease greater than distributor total');
      require(distributionRecord.total >= diff, 'decrease greater than distributionRecord total');
      total -= diff;
      records[beneficiary].total -= uint120(diff);
      token.safeTransfer(owner(), diff);
      // reduce voting power
      _burn(beneficiary, tokensToVotes(diff));
       // int value
+     adjusted[beneficiary] = diff;
    } else {
      // increasing claimable tokens
      total += diff;
      records[beneficiary].total += uint120(diff);
      // increase voting pwoer
      _mint(beneficiary, tokensToVotes(diff));

+     adjusted[beneficiary] = diff;
    }

    emit Adjust(beneficiary, amount);
  }
```

Then the adjusted value should be applied before re-initializing

```solidity
  function _executeClaim(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);
+   uint120 adjustedAmount = adjusted[beneficiary];

    // effects
+   if (records[beneficiary].total != (totalAmount - uint120(adjustedAmount))) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```

# 4. Re-initializing the beneficiary if the total has been updated doesn't mint any additional voting power to the beneficiary.
## Summary
Re-initializing the beneficiary if the total has been updated doesn't mint any additional voting power to the beneficiary.

## Vulnerability Detail
When executing a claim, the function re-initialize the beneficiary if the total has been updated, since the last time the beneficiary was initialized.
- The problem here is that the function initializes the benificary, but doesn't actually mint any additional voting power to it.

```solidity
  function _executeClaim(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);

    // effects
    if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount);

    // Checks
    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }
```
For example, take the PriceTierVestingSale_2_0 contract.
- The total depends on the purchased amount by the beneficiary.
- If the beneficiary purchase more tokens, his purchased amount increases.
- The beneficiary calls the claim function.
- The claim function will re-initalize his beneficiary to the current purchased amount but won't mint the additional voting power earned by the beneficiary depending on how much more tokens the beneficiary purchased.

## Impact
Loss of voting power duo to the contract initializing the beneficiary without minting any additional voting power to it.

## Code Snippet

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/Distributor.sol#L66

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/Distributor.sol#L47

## Tool used

Manual Review

## Recommendation
Hard to do a fix here as a lot of vesting contracts depend on the Distributor contract, and few of them have different logic. 

- E.g. the BasicDistributor contract doesn't have this issue, as the total is adjusted by the owner.

# 5. Wrong accounting of voting power minted to the beneficiary in most of the vesting contracts. 
## Summary
Wrong accounting of voting power minted to the beneficiary in most of the vesting contracts. 

## Vulnerability Detail
The function initializeDistributionRecord is open for users to initialize their distribution record to the current purchased amount. 

However this leads to the wrong accounting of the voting power, e.g take the following scenario:
1. User has 20 purchased tokens and calls the function to initialize his distribution record. As a result the function mints voting power corresponding the 20 tokens he purchased.
3. After some time the user purchases another 20 tokens, as a result the user has 40 purchased tokens now. 
4. The user calls the function to initialize his distribution record, but instead of only minting voting power for the additional 20 tokens he bought. It mints voting power corresponding the 40 tokens.
5. In the end duo to this problem, the user has voting power for 60 tokens, instead of the 40 tokens he purchased.

```solidity
  function initializeDistributionRecord(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) {
    _initializeDistributionRecord(beneficiary, getPurchasedAmount(beneficiary));
  }
```
```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

    // add voting power through ERC20Votes extension
    _mint(beneficiary, tokensToVotes(totalAmount));
  }
```

## Impact
Wrong accounting of voting power when initializing a beneficiary, can lead to the user receiving more voting power than the amount he purchased in the first place.

## Code Snippet

https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingSale_2_0.sol#L91

## Tool used

Manual Review

## Recommendation
Track the purchased amount with mapping and mint voting power only for the additional tokens the user purchased.

```solidity
  function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

+   uint256 purchasedAmount = lastPurchasedAmount[beneficiary];

    // add voting power through ERC20Votes extension
    _mint(beneficiary, tokensToVotes(purchasedAmount));

+   lastPurchasedAmount[beneficiary] = totalAmount;
  }
```
