## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-operatorgte-is-used-for-both-increase-or-decrease-cdp) |Operator.gte is used for both increase or decrease CDP | Medium |

## [M-01] Operator.gte is used for both increase or decrease CDP

## Impact

When collateral is increased or decreased via `_adjustCdp()`, within `_doOperation()`, there is a check for the expected collateral after the adjustment is made. This expected collateral is equal to OldCollateral +- CollateralChange multiplied by the buffer you have set. As each user passes `collValidationBufferBPS` when making an increment, it will be set to (e.g. 105% from the tests), which will limit the change to 5% above the expected value and ensure that their position is incremented to the maximum so far. But the same operator `Operator.gte` is used when decreasing. Which will completely remove the purpose of the check as it will always check if the user's **`expected value ≥ the actual amount`**. Which in the case of decrease should be the other way around as user should be allowed to set the **maximum amount** to be decreased and anything lower than that to be returned, but it isn't.

## Proof of Concept

Increase Example numbers:

- `oldCollateral = 100`
- `CollateralChange = 20`
- `NewCollateral should be 120`
- `collValidationBufferBPS = 105%`
- `expectedCollateral = 126`

And then 126 should be ≥ the changed collateral from `cdpInfo.coll` which will be around 120 but can also vary and be up to 126

```solidity
if (postCheckType == PostOperationCheck.cdpStats) {
    ICdpManagerData.Cdp memory cdpInfo = cdpManager.Cdps(checkParams.cdpId);

    _doCheckValueType(checkParams.expectedDebt, cdpInfo.debt);
    _doCheckValueType(checkParams.expectedCollateral, cdpInfo.coll);
    require(
        cdpInfo.status == checkParams.expectedStatus,
        "!LeverageMacroReference: adjustCDP status check"
    );
}
```

**≥** because this was the operator set for all collateral changes, no matter increment or decrement.

```jsx
expectedCollateral: CheckValueAndType({
        value:  _totalCollateral * _collValidationBuffer / BPS,
        operator: Operator.gte
    }),
```

And here will be the problem, when there is a decrease, the check will always pass because there will be no limit on the actual collateral change.

Decrease Example numbers:

- `oldCollateral = 100`
- `CollateralChange = 20`
- `NewCollateral should be 80`
- `collValidationBufferBPS = 105%`
- `expectedCollateral = 84`

here the comparison will be 84 ≥ changed collateral from `cdpInfo.coll`, which must be lower than that, which removes the idea of checking how much collateral is reduced, since the reduced collateral will never be small enough to break this case. But here is the other side of the coin that no matter how much collateral is actually reduced, this check is only for the Minimum Decrease Value and the user cannot limit how much they actually want removed. Since for both increment and decrement ≥ is used.

## Tools Used

Manual Review

## Recommended Mitigation Steps

For decrease operations, it will be better for users to pass lower than 100% buffer and then change the operation to ≤, thus making the Maximum Collateral to remove from position.