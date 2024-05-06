## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-first-depositor-inflation-attack-in-pendlepowerfarmtoken) | First depositor inflation attack in `PendlePowerFarmToken` | Medium |
| [M-02](#m-02-small-positions-will-not-pose-incentive-to-be-liquidated-bringing-bad-debt-to-the-poolmarket) | Small positions will not pose incentive to be liquidated bringing bad debt to the `poolMarket` | Medium |
| [M-03](#m-03-liquidator-will-brick-the-entire-pool-market-also-stealing-from-the-subsequent-depositors) | Liquidator will brick the entire pool market, also stealing from the subsequent depositors | Medium |

# [M-01] First depositor inflation attack in `PendlePowerFarmToken`

## Impact

In certain scenarios, shares of a subsequent depositor can be heavily reduced, losing a large amount of his deposited funds, the attacker can increase the right side of the `previewMintShares` by adding rewards for compounding.

That way victim can lose 6e17 of his assets for a deposit of 1e18.

## Proof of Concept

Let’s see how a first user, can grief a subsequent deposit and reduce his shares from the desired 1:1 ratio to 1:0000000000000005.

First, he needs to choose `PowerFarmToken` with no previous deposits. 

1. He calls `depositExactAmount` with 2 wei which will also call `syncSupply` → `_updateRewards` which is a key moment of the attack, this will make it possible `PowerFarmController::exchangeRewardsForCompoundingWithIncentive` to be called when performing the donation.
2. With 2 wei user will mint 2 shares, so totalSupply = 3, underlyingLpAssetsCurrent = 3
3. Then, the victim comes and tries to deposit 1e18 of assets, but is front ran by the first user calling `PowerFarmController::exchangeRewardsForCompoundingWithIncentive` → `addCompoundRewards` with 999999999999999996 that will increase the `totalLpAssetsToDistribute`, which is added to `underlyingLpAssetsCurrent` in the `_syncSupply` function, called from the modifier before the main functions.

> The attacker does not lose his deposit of 1e18 because it is converted to `rewardTokens` and sent to him, basically making the inflation free.
> 

[PendlePowerFarmController.sol#L53-L111](https://github.com/code-423n4/2024-02-wise-lending/blob/79186b243d8553e66358c05497e5ccfd9488b5e2/contracts/PowerFarms/PendlePowerFarmController/PendlePowerFarmController.sol#L53-L111)

```solidity
function exchangeRewardsForCompoundingWithIncentive(
      address _pendleMarket,
      address _rewardToken,
      uint256 _rewardAmount
  ) external syncSupply(_pendleMarket) returns (uint256) {
      CompoundStruct memory childInfo = pendleChildCompoundInfo[_pendleMarket];

      uint256 index = _findIndex(childInfo.rewardTokens, _rewardToken);

      if (childInfo.reservedForCompound[index] < _rewardAmount) {
          revert NotEnoughCompound();
      }

      uint256 sendingAmount = _getAmountToSend(_pendleMarket, _getTokensInETH(_rewardToken, _rewardAmount));

      childInfo.reservedForCompound[index] -= _rewardAmount;
      pendleChildCompoundInfo[_pendleMarket] = childInfo;

      _safeTransferFrom(_pendleMarket, msg.sender, address(this), sendingAmount);

      IPendlePowerFarmToken(pendleChildAddress[_pendleMarket]).addCompoundRewards(sendingAmount);//@audit inflate

      _safeTransfer(childInfo.rewardTokens[index], msg.sender, _rewardAmount);//@audit receive back + incentive

      emit ExchangeRewardsForCompounding(_pendleMarket, _rewardToken, _rewardAmount, sendingAmount);

      return sendingAmount;
  }
```

1. After that totalSupply = 3, underlyingLpAssetsCurrent = 1e18 - 1
2. Victim transaction is executed:

[PendlePowerFarmToken.sol#L452-L463](https://github.com/code-423n4/2024-02-wise-lending/blob/79186b243d8553e66358c05497e5ccfd9488b5e2/contracts/PowerFarms/PendlePowerFarmController/PendlePowerFarmToken.sol#L452-L463)

```solidity
function previewMintShares(uint256 _underlyingAssetAmount, uint256 _underlyingLpAssetsCurrent)
        public
        view
        returns (uint256)
    {
        return _underlyingAssetAmount * totalSupply() / _underlyingLpAssetsCurrent;
        // 1e18 * 3 / 1e18 - 1 = 2 
    }
```

Both attacker and victim have 1 share, because of the fee that is taken in the deposit function.

After victim deposit: 

totalSupply: 5, underlyingLpAssetsCurrent = 2e18 - 1

1. Then victim tries to withdraw all his shares - 1 (1 was taken as a fee on deposit)

[PendlePowerFarmToken.sol#L465-L476](https://github.com/code-423n4/2024-02-wise-lending/blob/79186b243d8553e66358c05497e5ccfd9488b5e2/contracts/PowerFarms/PendlePowerFarmController/PendlePowerFarmToken.sol#L465-L476)

```solidity
function previewAmountWithdrawShares(uint256 _shares, uint256 _underlyingLpAssetsCurrent)
        public
        view
        returns (uint256)
    {
        return _shares * _underlyingLpAssetsCurrent / totalSupply();
	        //1 * (2e18 - 1) / 5 = 399999999999999999 (0.3e18)
    }
```

User has lost 1e18 - 0.3e18 = 0.6e18 tokens.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Since there will be many `PowerFarmTokens` deployed, there is no way team to perform the first deposit for all of them. Possible mitigation will be to have minimum deposit amount for the first depositor in the `PendlePowerToken`, which will increase the cost of the attack exponentially and there will be no enough of reward tokens making the `exchangeRewardsForCompoundingWithIncentive` revert, due to insufficient amount.

Or just mint proper amount of tokens in the `initialize` function.

# [M-02] Small positions will not pose incentive to be liquidated bringing bad debt to the `poolMarket`

## Impact

There is no incentive for liquidators to liquidate small positions (even under $100), especially on Mainnet, this will result in bad debt accrued for the WiseLending system because no one will be willing to remove the position and `FEE_MANAGER` will not be able to split the bad debt between depositors.

## Proof of Concept

If we take a look at the current gas prices on Mainnet:

| Priority | Low | Average | High |
| --- | --- | --- | --- |
| Gwei | 75 | 75 | 80 |
| Price | $7 | $7 | $7,45 |

We can see the current transaction prices varying and in peak hours they can be up to 5 times more, without additional costs of liquidation logic execution.

That way if there are many borrow positions eligible for liquidation but with unsatisfactory incentives for liquidators (liquidator receives less than the cost of executing the liquidation) and they approach insolvency, receiving roughly $5,90 incentive for the $100 borrow, it will lead to big harm for the protocol since they will result in an unaccounted bad debt for the system, without a proper way to be mitigated. 

We can see in the code that there is no validation for the minimum borrowable amount, it has only to be enough to keep the position healthy. 

In fact, there is a check for the `minDepositEthValue` when depositing:

[WiseSecurity.sol#L1092-L1109](https://github.com/code-423n4/2024-02-wise-lending/blob/main/contracts/WiseSecurity/WiseSecurity.sol#L1092-L1109)

```solidity
function _checkMinDepositValue(
    address _token,
    uint256 _amount
)
    private
    view
    returns (bool)
{
    if (minDepositEthValue == ONE_WEI) {
        return true;
    }

    if (_getTokensInEth(_token, _amount) < minDepositEthValue) {
        revert DepositAmountTooSmall();
    }

    return true;
}
```

But it can do nothing to prevent this malicious action to be executed. 

> Important note is that positions can become insolvent unintentionally, by borrowers leaving dust amounts of borrowed tokens when closing their positions.
> 

For the borrow flow, there is no minimum borrowable amount.

[WiseLending.sol#L1553-L1581](https://github.com/code-423n4/2024-02-wise-lending/blob/main/contracts/WiseLending.sol#L1553-L1581)

```solidity
function _handleBorrowExactAmount(
    uint256 _nftId,
    address _poolToken,
    uint256 _amount,
    bool _onBehalf
)
    private
    returns (uint256)
{
    uint256 shares = calculateBorrowShares(
        {
            _poolToken: _poolToken,
            _amount: _amount,
            _maxSharePrice: true
        }
    );

    _coreBorrowTokens(
        {
            _caller: msg.sender,
            _nftId: _nftId,
            _poolToken: _poolToken,
            _amount: _amount,
            _shares: shares,
            _onBehalf: _onBehalf
        }
    );

    return shares;
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Similar to the deposit functions, consider adding `minBorrowEthValue` to prevent such scenarios, otherwise to keep the solvency of the whole `poolMarket` admins of Wise will have to liquidate such a positions, losing money in gas fees without receiving anything back.

# [M-03] Liquidator will brick the entire pool market, also stealing from the subsequent depositors

## Impact

When the pool lacks sufficient funds to fully compensate the liquidator, it will reset both `totalPool` and `totalLendingShares` to 0, but will increase the `lendingShares` of the liquidator by the difference.

This enables the liquidator to retrieve it later when the pool liquidity rises once again, either by `paybacks` or `lp deposits`. Consequently, when someone deposits into the pool, the liquidator can claim the remaining portion of their liquidation reward, preventing the depositor from withdrawing their full amount. As we can observe this creates a weird scenario when every next liquidity provider tokens are “stolen” from the previous withdrawer. 

This is extremely bad for the market because unlike borrowing, where you are sure that these funds will be back into the pool either by repaying or liquidation when the liquidator that has received surplus shares withdraws them it will lead to a domino effect and the last withdrawer for this market will bear all the cost risking up to 100% of his deposits to be stolen.

When `pureCollateral` of a user isn’t enough for the liquidator to seize, `_withdrawOrAllocateSharesLiquidation` is executed:

[WiseCore.sol#L460-L536](https://github.com/code-423n4/2024-02-wise-lending/blob/main/contracts/WiseCore.sol#L460-L536)

```solidity
function _withdrawOrAllocateSharesLiquidation(
    uint256 _nftId,
    uint256 _nftIdLiquidator,
    address _poolToken,
    uint256 _percentWishCollateral
)
    private
    returns (uint256)
{
 ...Code when withdrawAmount <= totalPoolToken
 

    uint256 totalPoolInShares = calculateLendingShares(
        {
            _poolToken: _poolToken,
            _amount: totalPoolToken,
            _maxSharePrice: false
        }
    );

    uint256 shareDifference = cashoutShares
        - totalPoolInShares;

    _coreWithdrawBare(
        _nftId,
        _poolToken,
        totalPoolToken,
        totalPoolInShares
    ); 
   
    _decreaseLendingShares(
        _nftId,
        _poolToken,
        shareDifference
    );

    _increasePositionLendingDeposit(
        _nftIdLiquidator,
        _poolToken,
        shareDifference
    );

    _addPositionTokenData(
        _nftIdLiquidator,
        _poolToken,
        hashMapPositionLending,
        positionLendTokenData
    );

    _removeEmptyLendingData(
        _nftId,
        _poolToken
    );
    
    //@audit after: 
    //totalPool = 0
    //totalDepositShares = 0
    //shares of liquidator = diff of cashout - total

    return totalPoolToken;
}
```

When assets of a pool are less than the amount that liquidator receives we can see that both 

- `lendingShares`
- `totalPool`

are being reset to 0.

And when borrowers payback, mappings that get increased are:

- `totalPool`
- `totalBorrowAmount`

[MainHelper.sol#L752-L788](https://github.com/code-423n4/2024-02-wise-lending/blob/main/contracts/MainHelper.sol#L752-L788)

```solidity
function _corePayback(
    uint256 _nftId,
    address _poolToken,
    uint256 _amount,
    uint256 _shares
)
    internal
{
    _updatePoolStorage(
        _poolToken,
        _amount,
        _shares,
        _increaseTotalPool,
        _decreasePseudoTotalBorrowAmount,
        _decreaseTotalBorrowShares
    );

    _decreasePositionMappingValue(
        userBorrowShares,
        _nftId,
        _poolToken,
        _shares
    );

    if (userBorrowShares[_nftId][_poolToken] > 0) {
        return;
    }

    _removePositionData({
        _nftId: _nftId,
        _poolToken: _poolToken,
        _getPositionTokenLength: getPositionBorrowTokenLength,
        _getPositionTokenByIndex: getPositionBorrowTokenByIndex,
        _deleteLastPositionData: _deleteLastPositionBorrowData,
        isLending: false
    });
}
```

This is not enough for a liquidator to redeem his difference because on withdraw the following mappings should hold enough values:

- `totalPool` - reset to 0 in liquidation
- `pseudoTotalPool`
- `totalDepositShares` - reset to 0 in liquidation

From there we can see in order liquidator to be able to redeem his shares he needs someone else to deposit his funds in the same market, we saw that `payback` won’t work. Then after `totalDepositShares` holds enough shares that satisfy the liquidator he can freely withdraw them “stealing” from the original depositor. Then when he wants to withdraw his tokens and exit the protocol it won’t be possible because previous user has redeemed tokens that belonged to him. 
Lets consider the example that deposits and withdrawals will be alternating. Then the scenario will continue to happen for each new deposit in the system being stolen from the previous person.

## Proof of Concept

Here is a gist containing test that can be placed in `test/extraFunctionality2.test.js`:

https://gist.github.com/AydoanB/1c8012f8e078d5b5de8981dee96a929e

For simplicity all the other tests can be removed, then in the terminal: `npm run test-extra2` will execute it. 

The main idea is to verify that when such type of liquidation occurs `totalPool` is zeroed and after Bob deposits in the system liquidator can redeem shares accrued from liquidation using Bob’s assets. 

After that when Bob wants to withdraw big part of his deposit `WiseCore::_coreWithdrawBare` will revert with underflow.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Finding an appropriate solution is indeed challenging. One approach could be to allow the liquidator to withdraw the remaining funds only when there is a substantial increase in liquidity, such as 5 times or 10 times the amount he has left to withdraw.