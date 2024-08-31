## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-loss-of-templegold-tokens-in-case-cross-chain-sender-is-account-abstraction-or-old-gnosis-wallet) |Loss of TempleGold tokens in case cross-chain sender is account abstraction or old gnosis wallet | High |
| [L-01](#l-01-templegoldstakingsetrewardduration-can-be-griefed-from-anyone-when-distributionstarter-is-unset) |`TempleGoldStaking::setRewardDuration` can be griefed from anyone when `distributionStarter` is unset | Low |

## [H-01] Loss of TempleGold tokens in case cross-chain sender is account abstraction or old gnosis wallet

## Summary

`TempleGold::send`, allows passing only `msg.sender` as a cross-chain recipient, which will result in lost or stolen tokens if executed by a smart contract, old gnosis wallet which is not deployed with a deterministic address, or account abstraction wallet due to the different address on the destination chain.

## Vulnerability Details

`send` function has a check which prevents Temple token holders from passing `to` address != than msg.sender, assuming on the front end there will not be even a possibility to give recipient address, this will result in lost tokens, in a similar to Wintermute fashion - [https://rekt.news/wintermute-rekt](https://rekt.news/wintermute-rekt/)

<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGold.sol#L290>

```solidity
src: TempleGold.sol#L290
function send(
        SendParam calldata _sendParam,
        MessagingFee calldata _fee,
        address _refundAddress
    ) external payable virtual override(IOFT, OFTCore) returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
        if (_sendParam.composeMsg.length > 0) { revert CannotCompose(); }
        /// cast bytes32 to address
        address _to = _sendParam.to.bytes32ToAddress();
        /// @dev user can cross-chain transfer to self
        if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }
...MORE CODE
```

We are sure that this is the recipient on the destination chain by simply looking at the natspec of the `SendParam`struct: <https://github.com/LayerZero-Labs/LayerZero-v2/blob/7aebbd7c79b2dc818f7bb054aed2405ca076b9d6/packages/layerzero-v2/evm/oapp/contracts/oft/interfaces/IOFT.sol#L12>

## Impact

Loss of bridged assets, due to always using msg.sender as a recipient of the cross-chain token send.

## Tools Used

Manual Review

## Recommendations

Consider allowing users to pass their own recipient, there are no security implications, such as blocked paths or reentrancies observed.

## [L-01] `TempleGoldStaking::setRewardDuration` can be griefed from anyone when `distributionStarter` is unset

## Summary

When `distributeRewards` can be called by anyone, no new reward duration from `setRewardDuration` or new vesting period from `setVestingPeriod` can be called, because it will always be griefed and `periodFinish` will be reset, reverting the transactions.

## Vulnerability Details

The issue will happen when `distributeRewards` is permissionless and everyone can call it, additionally setting a new vesting period and reward duration requires `rewardData::periodFinish` to be strictly less than the current `block.timestamp` :

```solidity
 function setVestingPeriod(uint32 _period) external override onlyElevatedAccess {
        if (_period < WEEK_LENGTH) { revert CommonEventsAndErrors.InvalidParam(); }
        // only change after reward epoch ends
        if (rewardData.periodFinish >= block.timestamp) { revert InvalidOperation(); }
        vestingPeriod = _period;
        emit VestingPeriodSet(_period);
    }

    /**
     * @notice Set reward duration
     * @param _duration Reward duration
     */
    function setRewardDuration(uint256 _duration) external override onlyElevatedAccess {
        // minimum reward duration
        if (_duration < WEEK_LENGTH) { revert CommonEventsAndErrors.InvalidParam(); }
        // only change after reward epoch ends
        if (rewardData.periodFinish >= block.timestamp) { revert InvalidOperation(); }
        rewardDuration = _duration;
        emit RewardDurationSet(_duration);
    }
```

But in distribute function we are always setting the `periodFinish` to current timestamp, which is the root cause of the issue:

```solidity
internally called by distributeRewards
function _notifyReward(uint256 amount) private {
      ...MORE CODE
      rewardData.lastUpdateTime = uint40(block.timestamp);
      rewardData.periodFinish = uint40(block.timestamp + rewardDuration);
  }
```

Here is a POC demonstrating that both functions will revert with `InvalidOperation`:

`forge test --match-test test_griefSetter`

```solidity
 function test_griefSetter() public {
        vm.startPrank(executor);
        staking.setMigrator(alice);
        uint256 _rewardDuration = 16 weeks;
        uint32 _vestingPeriod = uint32(_rewardDuration);
        _setVestingFactor(templeGold);
        _setVestingPeriod(_vestingPeriod);
        _setRewardDuration(_rewardDuration); 

          // bob stakes
        vm.startPrank(bob);
        deal(address(templeToken), bob, 1000 ether, true);
        _approve(address(templeToken), address(staking), type(uint).max);
        staking.stake(100 ether);
        vm.stopPrank();

        skip(2 days);
        staking.distributeRewards();

        vm.startPrank(executor);
        uint256 duration = 16 weeks;
        staking.setRewardDuration(duration);
    }
```

## Impact

Permanently blocking authorized entities from setting new duration and vesting periods.

## Tools Used

Manual review

## Recommendations

Simplest approach is to allow setting these values when periodFinish is equal to the current timestamp.