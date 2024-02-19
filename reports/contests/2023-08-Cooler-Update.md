## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-collateral-amount-will-be-inaccurate-if-the-exchange-rate-goes-up-a-lot) | Collateral amount will be inaccurate if the exchange rate goes up a lot | Medium |

# [M-01] Collateral amount will be inaccurate if the exchange rate goes up a lot

## Summary

User will put up much more collateral than needed if the exchange rate goes up a lot.

## Vulnerability Detail

Let's say we first borrow 2900DAI with 966,666,666,666,666,666 (0.96 gOHM) as collateral at a ratio of 3000e18 DAI/gOHM.

What if the ratio reaches 4000e18, the required collateral for the loan amount will be 726 201 712 328 767 122, isn't it necessary to return the difference to the borrower or reduce his debt? What if the price becomes very high e.g. 10000e18, the required collateral will now be 290,480,684,931,506,849, but there will be 966,666,666,666,666,666 as collateral, which is above than three times more. And when the loan becomes default, the lender will receive three times the amount they sent to the borrower. 

## Impact

High, because just before the default time, if the gOHM price jumps a lot, the borrower will lose the difference and the lender will get a lot more money.

## Code Snippet

Here the new collateral is calculated, but if its lower than the putted amount, its return 0.

```solidity
// File: Cooler/src/Cooler.sol
377: function newCollateralFor(uint256 loanID_) public view returns (uint256) {
    Loan memory loan = loans[loanID_];
    // Accounts for all outstanding debt (borrowed amount + interest).
    uint256 neededCollateral = collateralFor(
        loan.amount,
        loan.request.loanToCollateral
    );

    return
        neededCollateral > loan.collateral ?
        neededCollateral - loan.collateral :
        0;
}
```

https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Cooler.sol#L377-L389

```solidity
// File: Cooler/src/ClearingHouse.sol
178: uint256 newCollateral = cooler_.newCollateralFor(loanID_);
if (newCollateral > 0) {
    gOHM.transferFrom(msg.sender, address(this), newCollateral);
    gOHM.approve(address(cooler_), newCollateral);
}
```

https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L176-L180

https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Cooler.sol#L192-L217

## Tool used

Manual Review

## Recommendation

Consider returning the difference to the borrower and update the `loan.collateral` value, upon updating the `DAI/gOHM` ratio in the contract.
