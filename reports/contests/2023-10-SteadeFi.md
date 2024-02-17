## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-no-way-to-change-the-exchangerouter-implementation-in-case-its-updated) | No way to change the ExchangeRouter implementation in case it’s updated | Medium |
| [M-02](#m-02-wrong-hardcoded-pnl-factor-is-used-in-all-gmxvault-add-liquidity-operations) | Wrong hardcoded PnL factor is used in all GMXVault add liquidity operations | Medium |
| [M-03](#m-03-there-is-no-functionality-for-upgrade-or-migrate-to-new-gmx-implementations) | There is no functionality for upgrade or migrate to new GMX implementations | Medium |
| [L-01](#l-01-rebalance-may-occur-due-to-wrong-requirements-check) | Rebalance may occur due to wrong requirements check | Low |
| [L-02](#l-02-lack-of-events-for-critical-actions) | Lack of events for critical actions | Low |
| [L-03](#l-03-wrong-errors-are-used-for-reverts) | Wrong errors are used for reverts | Low |

# [M-01] No way to change the ExchangeRouter implementation in case it’s updated

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L100

## ## Summary

The current implementation of the GMXVault.sol sets the address of the ExchangeRouter(which is the core entry point of the SteadeFi for GMX) only in the constructor, which is stated as an integration issue in the GMX’s readme:
`• If using contracts such as the ExchangeRouter, Oracle or Reader do note that their addresses will change as new logic is added` 

https://github.com/gmx-io/gmx-synthetics#integration-notes 

## ## Vulnerability Details

The vulnerability lies in the fact that usually when a new ExchangeRouter is deployed from the GMX, the old router will not be used and funds have to be migrated to the newly deployed one.

Historically GMX is evolving daily and breaking changes in one of their core components will not be a surprise. 
The fact that this **issue** is stated explicitly in their README means that it will **definitely affect the existing markets**, but the thing that matters is that the whole balance of the market will be migrated to the new **ExchangeRouter,** which will make the existing GMXVaults that use the old router **hardcoded at the constructor** useless: `store.exchangeRouter`. 

That will result in the inability of all depositors of the GMXVault to withdraw their funds. 

## ## Impact

**Stuck** user funds due to the **old** ExchangeRouter not pointing to the right helper libraries: DepositUtils, WithdrawalUtils, and DataStore, which are all tied to the currently set ExchangeRouter in the constructor of the **GMXVault,** making it impossible for the users of the protocol to withdraw their funds.

## ## Tools Used

Manual, GMX GitHub

## ## Recommendations

Consider adding a function that allows changing the addresses of the core GMX contracts used:

```js
function updateExchangeRouterAddress(address exchangeRouter) external onlyOwner {
    require(exchangeRouter != address(0));
    _store.exchangeRouter = IExchangeRouter(exchangeRouter);

    emit ExchangeRouterUpdated(exchangeRouter);
}
```

# [M-02] Wrong hardcoded PnL factor is used in all GMXVault add liquidity operations

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L90-L101

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXManager.sol#L148

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/GMXOracle.sol#L244-L248

https://github.com/gmx-io/gmx-synthetics/blob/3b37cfa213471aac86344c352aa1636c67f41ca8/contracts/market/MarketUtils.sol#L136C14-L136C33

## Summary
Using a wrong hash when depositing into GMX Market will potentially stop all the deposits from GMXVaults, based on GMX’s deposit notes.

https://github.com/gmx-io/gmx-synthetics#deposit-notes

Which states:

• Deposits are not allowed above the MAX_PNL_FACTOR_FOR_DEPOSITS

## Vulnerability Details
The vulnerability lies in the fact that when either `GMXVault::deposit` and `GMXVault::rebalanceAdd` are called wrong pnlFactor (`MAX_PNL_FACTOR_FOR_WITHDRAWALS`) will be passed to the oracle function `GMXOracle::getLpTokenValue` which is intended to fetch the price of the market token when deposit and withdrawal functions are called. 
## Impact
As you can see in every time when the minimum market slippage amount is calculated pnl factor for withdrawals will be used:

```solidity
src: GMXDeposit.sol#L90-L101

if (dp.token == address(self.lpToken)) {
      // If LP token deposited
      _dc.depositValue = self.gmxOracle.getLpTokenValue(
        address(self.lpToken),
        address(self.tokenA),
        address(self.tokenA),
        address(self.tokenB),
        false, //@audit when depositing this should be set to true
        false
      )
      * dp.amt
      / SAFE_MULTIPLIER;
	}
```

```solidity
src: GMXOracle.sol#L234-L257

function getLpTokenValue(
    address marketToken,
    address indexToken,
    address longToken,
    address shortToken,
    bool isDeposit,
    bool maximize
  ) public view returns (uint256) {
    bytes32 _pnlFactorType;

    if (isDeposit) {
      _pnlFactorType = keccak256(abi.encode("MAX_PNL_FACTOR_FOR_DEPOSITS"));
    } else {
      _pnlFactorType = keccak256(abi.encode("MAX_PNL_FACTOR_FOR_WITHDRAWALS"));
    }

    (int256 _marketTokenPrice,) = getMarketTokenInfo(
      marketToken,
      indexToken,
      longToken,
      shortToken,
      _pnlFactorType,
      maximize
    );
```

Problem occurs when both `MAX_PNL_FACTOR_FOR_DEPOSITS` and `MAX_PNL_FACTOR_FOR_WITHDRAWALS` have different values.

There are 2 possible scenarios:
1. `MAX_PNL_FACTOR_FOR_WITHDRAWALS` is less than `MAX_PNL_FACTOR_FOR_DEPOSITS` 

In this case, when the user wants to deposit the maximum allowed amount based on `MAX_PNL_FACTOR_FOR_DEPOSITS` transaction will most likely revert because there will be a different price of lpToken returned from the GMXOracle called with the pnlFactor = `MAX_PNL_FACTOR_FOR_WITHDRAWALS`, instead of the one for **deposits.**

1. `MAX_PNL_FACTOR_FOR_WITHDRAWALS` is more than `MAX_PNL_FACTOR_FOR_DEPOSITS` 

In this case, GMXMarket’s Reader contract will return better price of the market token for the user, allowing him to deposit more than the actual value of `MAX_PNL_FACTOR_FOR_DEPOSIT`.
## Tools Used

## Recommendations
Change the isDeposit to **true** argument passed in the following functions: `GMXDeposit::deposit` and `GMXRebalance::rebalanceAdd`

```diff
if (dp.token == address(self.lpToken)) {
      // If LP token deposited
      _dc.depositValue = self.gmxOracle.getLpTokenValue(
        address(self.lpToken),
        address(self.tokenA),
        address(self.tokenA),
        address(self.tokenB),
-       false,
+       true,
        false
      )
      * dp.amt
      / SAFE_MULTIPLIER;
	}
```

# [M-03] There is no functionality for upgrade or migrate to new GMX implementations

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L100-L104

## Summary

In the current implementation of a GMXVault, there is no functionality for migration or upgrading if any of the GMX external contracts receive new implementations or if any of its associated markets migrate their tokens to another location.

## Vulnerability Details

As it stands, the **`GMXVault`** owner has initially set all the key external contracts, including:

- ExchangeRouter
- Router
- DepositVault
- WithdrawalVault
- RoleStore

However, there is currently no mechanism in place to update these contracts. Additionally, if, at any point in the future, GMX decides to move reserves to different vaults using new addresses, this contract will result in stuck of funds because it won't reflect the latest versions of these contracts.

## Impact

Not updating important external contracts will lead to old implementations and is error-prone.

## Tools Used

Manual

## Recommendations

It would be advisable to include logic that allows for the updating of these addresses and also logic to facilitate the migration of tokens in response to future changes. This proactive approach will enhance the flexibility and adaptability of the **`GMXVault`** contract and ensure it remains compatible with any modifications.

# [L-01] Rebalance may occur due to wrong requirements check

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L268-L293

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L312-L332

## Summary

Before a rebalance can occur, checks are implemented to ensure that **`delta`** and **`debtRatio`** remain within their specified limits. However, it's important to note that the check in **`GMXChecks::beforeRebalanceChecks`** ignores the scenario where these values are equal to any of their limits.

## Vulnerability Details

In the current implementation of the **`GMXRebalance::rebalanceAdd`** function, it first calculates the current values of **`debtRatio`** and **`delta`** before making any changes. Subsequently, the **`beforeRebalanceChecks`** function, checks if these values meet the requirements for a rebalance to occur. These requirements now dictate that both **`debtRatio`** and **`delta`** must be either ≥ to the **`UpperLimit`**, or ≤ to the **`LowerLimit`** for a rebalance to take place.

```js
function beforeRebalanceChecks(
  GMXTypes.Store storage self,
  GMXTypes.RebalanceType rebalanceType
) external view {
  if (
    self.status != GMXTypes.Status.Open &&
    self.status != GMXTypes.Status.Rebalance_Open
  ) revert Errors.NotAllowedInCurrentVaultStatus();

  // Check that rebalance type is Delta or Debt
  // And then check that rebalance conditions are met
  // Note that Delta rebalancing requires vault's delta strategy to be Neutral as well
  if (rebalanceType == GMXTypes.RebalanceType.Delta && self.delta == GMXTypes.Delta.Neutral) {
    if (
      self.rebalanceCache.healthParams.deltaBefore < self.deltaUpperLimit &&
      self.rebalanceCache.healthParams.deltaBefore > self.deltaLowerLimit
    ) revert Errors.InvalidRebalancePreConditions();
  } else if (rebalanceType == GMXTypes.RebalanceType.Debt) {
    if (
      self.rebalanceCache.healthParams.debtRatioBefore < self.debtRatioUpperLimit &&
      self.rebalanceCache.healthParams.debtRatioBefore > self.debtRatioLowerLimit
    ) revert Errors.InvalidRebalancePreConditions();
  } else {
     revert Errors.InvalidRebalanceParameters();
  }
}
```

Suppose a rebalance is successful. In the **`afterRebalanceChecks`** section, the code verifies whether both **`delta`** and **`debtRatio`** are greater than the **`UpperLimit`** or less than the **`LowerLimit`**. This confirmation implies that these limits are indeed inclusive, meaning that the correct interpretation of these limits should be that **`LowerLimit ≤ actualValue ≤ UpperLimit`**. On the other hand, this also indicates that for a rebalancing to occur, the values of **`deltaBefore`** and **`debtRatioBefore`** need to be outside their limits, i.e., **`delta`** should be greater than **`Upper`** or less than **`Lower`**. However, in the current implementation, if these values are equal to the limit, a rebalance may still occur, which violates the consistency of the **`afterRebalanceChecks`** function, thus indicating that these limits are inclusive. Consequently, a value equal to the limit needs to be treated as valid and not be able to trigger a rebalance.

```js
function afterRebalanceChecks(
  GMXTypes.Store storage self
) external view {
  // Guards: check that delta is within limits for Neutral strategy
  if (self.delta == GMXTypes.Delta.Neutral) {
    int256 _delta = GMXReader.delta(self);

    if (
      _delta > self.deltaUpperLimit ||
      _delta < self.deltaLowerLimit
    ) revert Errors.InvalidDelta();
  }

  // Guards: check that debt is within limits for Long/Neutral strategy
  uint256 _debtRatio = GMXReader.debtRatio(self);

  if (
    _debtRatio > self.debtRatioUpperLimit ||
    _debtRatio < self.debtRatioLowerLimit
  ) revert Errors.InvalidDebtRatio();
}
```

Imagine the case when **`delta`** or **`debtRatio`** is equal to any of its limits; a rebalance will occur. However, on the other hand, these values are valid because they are inclusively within the limits.

## Impact

In such a scenario, the system might incorrectly trigger a rebalance of the vault, even when **`delta`** or **`debtRatio`** is precisely within the established limits, thus potentially causing unintended rebalancing actions.

## Tools Used

Manual

## Recommendations

Consider a strict check to determine if **`delta`** or **`debtRatio`** is strictly within its limits, including scenarios where they are equal to any of its limits. In such cases, the code should ensure that a rebalance does not occur when these values are precisely at the limit.

```diff
function beforeRebalanceChecks(
    GMXTypes.Store storage self,
    GMXTypes.RebalanceType rebalanceType
  ) external view {
    if (
      self.status != GMXTypes.Status.Open &&
      self.status != GMXTypes.Status.Rebalance_Open
    ) revert Errors.NotAllowedInCurrentVaultStatus();

    // Check that rebalance type is Delta or Debt
    // And then check that rebalance conditions are met
    // Note that Delta rebalancing requires vault's delta strategy to be Neutral as well
    if (rebalanceType == GMXTypes.RebalanceType.Delta && self.delta == GMXTypes.Delta.Neutral) {
      if (
-       self.rebalanceCache.healthParams.deltaBefore < self.deltaUpperLimit &&
-       self.rebalanceCache.healthParams.deltaBefore > self.deltaLowerLimit
+       self.rebalanceCache.healthParams.deltaBefore <= self.deltaUpperLimit &&
+       self.rebalanceCache.healthParams.deltaBefore >= self.deltaLowerLimit
      ) revert Errors.InvalidRebalancePreConditions();
    } else if (rebalanceType == GMXTypes.RebalanceType.Debt) {
      if (
-       self.rebalanceCache.healthParams.debtRatioBefore < self.debtRatioUpperLimit &&
-       self.rebalanceCache.healthParams.debtRatioBefore > self.debtRatioLowerLimit
+       self.rebalanceCache.healthParams.debtRatioBefore <= self.debtRatioUpperLimit &&
+       self.rebalanceCache.healthParams.debtRatioBefore >= self.debtRatioLowerLimit
      ) revert Errors.InvalidRebalancePreConditions();
    } else {
       revert Errors.InvalidRebalanceParameters();
    }
  }
```

# [L-02] Lack of events for critical actions

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L161

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXCompound.sol#L35

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L72

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXManager.sol#L225

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXManager.sol#L244

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXRebalance.sol#L35

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXRebalance.sol#L149

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L334

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWithdraw.sol#L231

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L23

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L72

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L114

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L129

## Summary

There are functions that don’t emit events.

## Vulnerability Details

Functions: 

`GMXDeposit::processDepositFailure`

`GMXCompound::compound`

`GMXEmergency::emergencyResume`

`GMXManager::borrow`

`GMXManager::repay`

`GMXRebalance::rebalanceAdd`

`GMXRebalance::rebalanceRemove`

`GMXVault::mintFee`

`GMXWithdraw::processWithdrawFailure`

`GMXWorker::addLiquidity`

`GMXWorker::removeLiquidity`

`GMXWorker::swapExactTokensForTokens`

`GMXWorker::swapTokensForExactTokens`

## Impact

External users, frontend, or blockchain monitoring won’t announce for these critical functions.

## Tools Used

Manual

## Recommendations

Add events where possible for critical operations.

# [L-03] Wrong errors are used for reverts

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L68-L69

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L74-L75

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L351-L352

## Summary

There are checks that revert with wrong errors

## Vulnerability Details

Reverts:

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L68-L69

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L74-L75

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXChecks.sol#L351-L352

```solidity
File: contracts/strategy/gmx/GMXChecks.sol

// Should be Errors.EmptyDepositAmount
68: if (self.depositCache.depositParams.amt == 0)
      revert Errors.InsufficientDepositAmount();

// Should be Errors.EmptyDepositAmount
74: if (depositValue == 0)
      revert Errors.InsufficientDepositAmount();

// Should be Errors.EmptyDepositAmount
351: if (self.compoundCache.depositValue == 0)
      revert Errors.InsufficientDepositAmount();
```

## Impact

This can lead to user confusion as they won't receive the accurate revert reason.

## Tools Used

Manual

## Recommendations

Consider using `Errors.EmptyDepositAmount` for the provided cases.