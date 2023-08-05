# CodeHawks Escrow Contract

## What is CodeHawks Escrow?
Codehawks escrow is project meant to enable smart contract auditors (sellers) and smart contract protocols looking for audits (buyers) to connect using a credibly neutral option, with optional arbitration.

## Audit scope
- [src/Escrow.sol](https://github.com/Cyfrin/2023-07-escrow/blob/main/src/Escrow.sol)
- [src/EscrowFactory.sol](https://github.com/Cyfrin/2023-07-escrow/blob/main/src/EscrowFactory.sol)

## Issues found by Yuki
| Severity | Title | Count |
|:--|:--|:--:|
| High | Wrong accounting occurs, as a result the seller will get less amount of pay than both parties agreed on when the escrow was created. | 1 |
| Medium | In some cases buyers won't be able to get their funds back without putting their trust on the sellers to voluntarily return them back. | 2 |

# 1. Wrong accounting occurs, as a result the seller will get less amount of pay than both parties agreed on when the escrow was created.

## Summary
Wrong accounting occurs, as a result the seller will get less amount of pay than both parties agreed on when the escrow was created.

## Vulnerability Details
Before the creation of the escrow contract, both parties the buyer (protocol) and the seller (auditor) agree on certain price that will be paid to the auditor for performing the private audit.

Currently the escrow contract checks whether the needed amount of tokens were transferred to it.
```solidity
if (tokenContract.balanceOf(address(this)) < price) revert Escrow__MustDeployWithTokenBalance();
```

However an interesting scenario is not take into account, which will slightly reduce the price paid to the auditor for his security services. Which breaks the initial agreement between the two parties on the price paid for the private audit. 

Before the escrow is created, both parties agree on the arbiter details. The arbiter is trusted role given to a person, which confers with both parties off-chain when dispute needs to be resolved.

Issue scenario:
1. Buyer and seller agrees for an private audit and create an escrow contract. The buyer provides the needed amount of tokens to pay the agreed price for the audit.
2. Seller is done with the private audit and sends his security report to the buyer.
3. Buyer is not happy with the security report and doesn't want to pay the agreed price to the seller, so he calls initiateDispute.
4. The arbiter consults with both parties, and approves that the seller deserves the price as the private audit was performed and security report was provided to the buyer.
5. The arbiter makes his decisions and resolves the dispute, by not refunding any tokens to the buyer.
6. The problem here is that by resolving the dispute, a significant share of the price can be paid as fee to the arbiter. As a result the agreed price for the private audit won't be paid to the seller.

```solidity
    function resolveDispute(uint256 buyerAward) external onlyArbiter nonReentrant inState(State.Disputed) {
        uint256 tokenBalance = i_tokenContract.balanceOf(address(this));
        uint256 totalFee = buyerAward + i_arbiterFee; // Reverts on overflow
        if (totalFee > tokenBalance) {
            revert Escrow__TotalFeeExceedsBalance(tokenBalance, totalFee);
        }

        s_state = State.Resolved;
        emit Resolved(i_buyer, i_seller);

        if (buyerAward > 0) {
            i_tokenContract.safeTransfer(i_buyer, buyerAward);
        }
        if (i_arbiterFee > 0) {
            i_tokenContract.safeTransfer(i_arbiter, i_arbiterFee);
        }
        tokenBalance = i_tokenContract.balanceOf(address(this));
        if (tokenBalance > 0) {
            i_tokenContract.safeTransfer(i_seller, tokenBalance);
        }
    }
```

Not receiving the agreed price for performing the private audit, can reduce the initiative of auditors to provide their security services on the platform.

## Impact
Potential loss of funds for the seller, and reduce initiative of auditors to provide their security services on the platform.

## Tools Used 
Manual review.

## Recommend Mitigation
On escrow creation, the buyer should provide an amount of tokens which should cover both the price paid to the auditor and arbiter fee.

This way the agreed price for the private audit will be paid to the auditor, incase when resolve dispute is made in favor of the seller.

```solidity
    constructor(
        uint256 price,
        IERC20 tokenContract,
        address buyer,
        address seller,
        address arbiter,
        uint256 arbiterFee
    ) {
        if (address(tokenContract) == address(0)) revert Escrow__TokenZeroAddress();
        if (buyer == address(0)) revert Escrow__BuyerZeroAddress();
        if (seller == address(0)) revert Escrow__SellerZeroAddress();
        if (arbiterFee >= price) revert Escrow__FeeExceedsPrice(price, arbiterFee);
+       if (tokenContract.balanceOf(address(this)) < price + arbiterFee) revert Escrow__MustDeployWithTokenBalance();
        i_price = price;
        i_tokenContract = tokenContract;
        i_buyer = buyer;
        i_seller = seller;
        i_arbiter = arbiter;
        i_arbiterFee = arbiterFee;
    }
```

In a case when the receipt was confirmed, the additional amount of tokens provided for the arbiter fee should be returned back to the buyer. On escrow creation the buyer provided both tokens for the price and the arbiter fee, so everything should work fine.

```solidity
    function confirmReceipt() external onlyBuyer inState(State.Created) {
        s_state = State.Confirmed;
        emit Confirmed(i_seller);

+      i_tokenContract.safeTransfer(i_buyer, i_arbiterFee);  

       i_tokenContract.safeTransfer(i_seller, i_tokenContract.balanceOf(address(this)));   
 }

```

# 2. In some cases buyers won't be able to get their funds back without putting their trust on the sellers to voluntarily return them back.

## Summary
In some cases buyers won't be able to get their funds back without putting their trust on the sellers to voluntarily return them back.

## Vulnerability Details
Currently the contract allows deploying an escrow with address(0) as arbiter. 

Not using an arbiter could be a result by either of the parties having trust issues towards a third person to allocate the right amount of tokens when dispute is resolved. Thats why there is an option to deploy an escrow contract with address(0) as arbiter, this indicates that arbiter won't be used and initiateDispute can't be made.

However there is a big downside of not using arbiter, which the current system doesn't predict:

1. Buyer and seller make an agreement for a private audit and deploys a new escrow contract with address(0), as for some reason either of the parties have a trust issue towards third party. The two parties schedule a week on which the private audit will be done.
2. The scheduled time comes and for some reason the seller doesn't want to do the private audit anymore. It could be that the auditor simply has something else to do or just a protocol with better offer showed up.
3. As there is no arbiter, a dispute can't be initiated to return back the tokens to the buyer.
4. The only available way to send the funds outside of the escrow, would be to call confirmReceipt and send the amount of tokens the contract has to the seller.
5. In the end there is no guarantee that the seller will voluntarily return the tokens back to the buyer. 

```solidity
    function initiateDispute() external onlyBuyerOrSeller inState(State.Created) {
        if (i_arbiter == address(0)) revert Escrow__DisputeRequiresArbiter();
        s_state = State.Disputed;
        emit Disputed(msg.sender);
    }
```

## Impact
In a case when an arbiter is not used in the escrow and the private audit is not performed. The only way for the buyer to receive his funds back from the escrow would be to put trust on the seller to voluntarily return them back.

## Tools Used 
Manual review.

## Recommend Mitigation
The only recommended fix l could've think of is to not allow an escrow to be deployed without an arbiter. 

```solidity
    constructor(
        uint256 price,
        IERC20 tokenContract,
        address buyer,
        address seller,
        address arbiter,
        uint256 arbiterFee
    ) {
        if (address(tokenContract) == address(0)) revert Escrow__TokenZeroAddress();
        if (buyer == address(0)) revert Escrow__BuyerZeroAddress();
        if (seller == address(0)) revert Escrow__SellerZeroAddress();
+       if (arbiter == address(0)) revert Escrow__ArbiterZeroAddress();
        if (arbiterFee >= price) revert Escrow__FeeExceedsPrice(price, arbiterFee);
        if (tokenContract.balanceOf(address(this)) < price) revert Escrow__MustDeployWithTokenBalance();
        i_price = price;
        i_tokenContract = tokenContract;
        i_buyer = buyer;
        i_seller = seller;
        i_arbiter = arbiter;
        i_arbiterFee = arbiterFee;
    }
```
