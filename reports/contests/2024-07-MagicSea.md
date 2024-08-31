## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-votervote-will-be-always-reverting-in-case-briberewarder-is-registered) |Voter::vote will be always reverting in case BribeRewarder is registered | High |
| [H-02](#h-02-briberewarder-unclaimed-tokens-remain-locked-due-to-missing-functionality) |BribeRewarder unclaimed tokens remain locked due to missing functionality | High |
| [H-03](#h-03-users-retain-their-voting-power-even-when-their-lock-expires) |Users retain their voting power even when their lock expires | High |
| [H-04](#h-04-users-cannot-claim-bribe-rewards-2-periods-after-_lastvotingperiod) |Users cannot claim bribe rewards 2 periods after _lastVotingPeriod | High |
| [M-01](#m-01-malicious-user-can-deploy-5-briberewarders-for-uint-max-periods-of-given-pool) |Malicious user can deploy 5 BribeRewarders for uint max periods of given pool | Medium |
| [M-02](#m-02-mlumstaking-user-can-frontrun-new-reward-tokens-and-harvest-immediatelly) |MlumStaking user can frontrun new reward tokens and harvest immediatelly | Medium |
| [M-03](#m-03-anyone-can-call-addtoposition) |Anyone can call addToPosition  | Medium |
| [M-04](#m-04-mlumstakingaddtoposition-doesnt-consider-fee-on-transfer-tokens-unfairly-calculating-the-avgduration) |`MlumStaking::addToPosition` doesn’t consider fee-on-transfer tokens, unfairly calculating the avgDuration | Medium |
| [M-05](#m-05-harvestpositionsto-cannot-be-used-as-intended) |harvestPositionsTo cannot be used as intended | Medium |

## [H-01] Voter::vote will be always reverting in case BribeRewarder is registered

## Summary

Due to wrong NFT owner checks, voting in pools with active `BribeRewarders` will be impossible.

## Vulnerability Detail

The flow of user voting from `Voter` for pool with registered BriberRewarder is as follows: Voter::vote → BribeRewarder::deposit. `msg.sender` in the BribeRewarder will be the `Voter` contract.

The problem lies in the `_modify` implementation as it checks if the `msg.sender` is the owner of this given `tokenId`, but the owner is the user itself, not the Voter contract and tx will revert:

```solidity

src: BribeRewarder.sol
function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
    private
    returns (uint256 rewardAmount)
{
    if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
        revert BribeRewarder__NotOwner();
    }
..MORE CODE
}
```

```solidity
src: Voter.sol
function ownerOf(uint256 tokenId, address account) external view returns (bool) {
    return _mlumStaking.ownerOf(tokenId) == account;
}
```

This issue can be caused both intentionally or not, by simply registering any BribeRewarder for the given pool and period.

## Impact

Full dos of the voting functionality in case there are active `BribeRewarders` registered for the given pool

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L264-L266

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L402-L404

## Tool used

Manual Review

## Recommendation

Move the token ownership check to `BribeRewarder::claim`

## [H-02] BribeRewarder unclaimed tokens remain locked due to missing functionality

## Summary

According to the MagicSea [documentation](https://docs.magicsea.finance/protocol/magic/magic-lum-voting#bribes), the **unclaimed rewards** from BribeRewarders should be able to be claimed by the owner, but looking at the available functionality we can clearly see that there is no such functionality available.

## Vulnerability Detail

> **Bribes**
> 
> 
> Bribes as an additional incentive to vote can be claimed 24-48 hours after an epoch has ended. Voters can claim the rewards until the next epoch is ended. Unclaimed rewards will be sent back to the briber.
> 

Not full reward utilisation is a valid scenario and it can happen due to various factors:

- not enough incentive for the users to vote for the pool and opt-in for bribe rewards
- inactivity of users, since rewards can be claimed up to 48 hours after last period ended
- initial funding provided is more than the `totalAmount`, as the check is not strict:

```solidity
function _bribe(uint256 startId, uint256 lastId, uint256 amountPerPeriod) internal {
        uint256 totalAmount = _calcTotalAmount(startId, lastId, amountPerPeriod);

        uint256 balance = _balanceOfThis(_token());

        if (balance < totalAmount) revert BribeRewarder__InsufficientFunds();
        ..MORE CODE
    }
```

All these can lead to scenarios when the balance of the `BribeRewarder` is not zero after the designated claim period, the issue is that the owner cannot claim the leftover tokens, since there is no functionality available, since the only place where `_safeTransferTo` is called is the `_modify` function that is tied only for tokenIds that have voted for the pool.

## Impact

Unclaimed rewards will be forever locked in the bribe contract, the impact is even higher because single bribe is for single set of periods, after `_lastVotingPeriod` passes new contract must be deployed and funded, thus there is no way these locked tokens even to be used.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol

## Tool used

Manual Review

## Recommendation

Add owner-restricted sweep function to claim the locked tokens.

## [H-03] Users retain their voting power even when their lock expires

## Summary

`lockDuration` check in the `Voter` contract is not sufficient and will enable users with expired locks to retain their voting power forever.

## Vulnerability Detail

As mentioned in `MagicSea` [docs](https://docs.magicsea.finance/protocol/magic/magic-lum-voting):

> The overall lock needs to be longer than 90 days and the remaining lock period needs to be longer than the epoch time.
> 

The `lockDuration` should be at least longer than the current period duration, but the issue is that if users only harvest their positions without modifying them they will be able to vote forever, since it only check for the overall duration, not the remaining lock time of a position:

```solidity
function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
...MORE CODE
    // check if _minimumLockTime >= initialLockDuration and it is locked
    if (_mlumStaking.getStakingPosition(tokenId).initialLockDuration < _minimumLockTime) {
        revert IVoter__InsufficientLockTime();
    }
    if (_mlumStaking.getStakingPosition(tokenId).lockDuration < _periodDuration) {
        revert IVoter__InsufficientLockTime();
    }
}
```

The valid requirement is the `initialLockDuration` to be longer than 90 days, the second check doesn’t properly check how much time there is until the position is unlocked. As we mentioned this makes each position with `lockDuration` > 90 days, to be able to vote forever.

## Impact

Users with expired positions will be able to vote.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L175-L177

## Tool used

Manual Review

## Recommendation

Modify the check to validate the remaining lock duration, instead of the overall lock duration:

```solidity
if ((_mlumStaking.getStakingPosition(tokenId).startLockTime + _mlumStaking.getStakingPosition(tokenId).lockDuration) - block.timestamp < _periodDuration) {
        revert IVoter__InsufficientLockTime();
    }
```

## [H-04] Users cannot claim bribe rewards 2 periods after _lastVotingPeriod

## Summary

Users cannot claim Bribe rewards 2 periods after the `_lastVotingPeriod` of his BribeRewarder pass.

## Vulnerability Detail

Each Bribe Rewarder has a set of Voter periods and rewards are based on them. Rewarder can be registered with `startId` to `endId` only for future Voter periods.

Then the user associated with that rewarder can `claim()` rewards for each period. But due to a wrong implementation of `claim()`, if they don't claim by `BribeRewarder._lastVotingPeriod() + 2 periods`, they will never be able to claim their rewards.

This will happen because of 2 points:

- Bribe Rewarders have a specific `endPeriod`, but the Voter will continue to start new periods even after passing the specific Bribe Rewarder `endPeriod`.
- `claim()` gets the `latestFinishedPeriod` of the Voter, but when this is called after BribeRewarder's `endId`, `claim()` will loop more times than periods in the Rewarder, which will return out of bounds when indexing `_rewards`.

```solidity
function claim(uint256 tokenId) external override {
    uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();

    uint256 totalAmount;

    // calc emission per period cause every period can every other durations
    for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
        totalAmount += _modify(i, tokenId, 0, true);
    }

    emit Claimed(tokenId, _pool(), totalAmount);
}
```

```solidity
function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
    private
    returns (uint256 rewardAmount)
{
    if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
        revert BribeRewarder__NotOwner();
    }

    // extra check so we dont calc rewards before starttime
    (uint256 startTime,) = IVoter(_caller).getPeriodStartEndtime(periodId);
    if (block.timestamp <= startTime) {
        _lastUpdateTimestamp = startTime;
    }

    RewardPerPeriod storage reward = _rewards[_indexByPeriodId(periodId)]; // @audit - revert with out of bounds
    Amounts.Parameter storage amounts = reward.userVotes;
    Rewarder2.Parameter storage rewarder = reward.rewarder;
```

*Note: It will **revert** after `endPeriod + 2`, due to a wrong for loop in `bribe()` that adds 1 more row inside `_rewards`, but that's not a problem in this case, it just increases the claim window by 1 period.*

## Impact

Bribe rewards cannot be claimed if the user misses to call `BribeRewarder.claim()` between `_startVotingPeriod` and `_lastVotingPeriod+2`.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L154

## Tool used

Manual Review

## Recommendation

Limit `endPeriod` inside `claim()` to `_lastVotingPeriod`.

```diff
function claim(uint256 tokenId) external override {
    uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();
+   endPeriod = endPeriod >= _lastVotingPeriod ? _lastVotingPeriod : endPeriod;
    

    uint256 totalAmount;

    // calc emission per period cause every period can every other durations
    for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
        totalAmount += _modify(i, tokenId, 0, true);
    }

    emit Claimed(tokenId, _pool(), totalAmount);
}
```

## [M-01] Malicious user can deploy 5 BribeRewarders for uint max periods of given pool

## Summary

Due to the permissionless nature of BribeRewarder registration, malicious users can block the bribe registration from pool owners, preventing them from providing additional incentives for voting.

## Vulnerability Detail

Everyone can deploy a new `BribeRewarder` contract with any token he wants, register for as many periods, and fund as much as he wants, even 0. 

Additionally, there is a limit on the BribeRewarders that can register for the pool per period. 

Knowing that everyone can deploy 5 `BribeRewarders` per pool and bribe with all of them for the periods starting from `_currentVotingPeriod` + 1 to uint256 max. 

```solidity
function bribe(uint256 startId, uint256 lastId, uint256 amountPerPeriod) public onlyOwner {
      _bribe(startId, lastId, amountPerPeriod);
  }

function _bribe(uint256 startId, uint256 lastId, uint256 amountPerPeriod) internal {
      _checkAlreadyInitialized();
      if (lastId < startId) revert BribeRewarder__WrongEndId();
      if (amountPerPeriod == 0) revert BribeRewarder__ZeroReward();

      IVoter voter = IVoter(_caller);

      if (startId <= voter.getCurrentVotingPeriod()) {
          revert BribeRewarder__WrongStartId();
      }

      uint256 totalAmount = _calcTotalAmount(startId, lastId, amountPerPeriod);

      uint256 balance = _balanceOfThis(_token());

      if (balance < totalAmount) revert BribeRewarder__InsufficientFunds();

      _startVotingPeriod = startId;
      _lastVotingPeriod = lastId;
      _amountPerPeriod = amountPerPeriod;

      // create rewads per period
      uint256 bribeEpochs = _calcPeriods(startId, lastId);
      for (uint256 i = 0; i <= bribeEpochs; ++i) {
          _rewards.push();
      }

      _lastUpdateTimestamp = block.timestamp;

      IVoter(_caller).onRegister();

      emit BribeInit(startId, lastId, amountPerPeriod);
  }
```

As we see `amountPerPeriod` is a parameter that is passed by the user and can be 0. 

The problem will happen in `Voter::onRegister` and it is due to the fact that there is a max limit on the bribes:

```solidity
/**
   * @dev bribe rewarder registers itself
   * TODO check if rewarder is from allowed rewarderFactory
   */
  function onRegister() external override {
      IBribeRewarder rewarder = IBribeRewarder(msg.sender);

      _checkRegisterCaller(rewarder);

      uint256 currentPeriodId = _currentVotingPeriodId;
      (address pool, uint256[] memory periods) = rewarder.getBribePeriods();
      for (uint256 i = 0; i < periods.length; ++i) {
          // TODO check if rewarder token + pool  is already registered

          require(periods[i] >= currentPeriodId, "wrong period");
          require(_bribesPerPriod[periods[i]][pool].length + 1 <= Constants.MAX_BRIBES_PER_POOL, "too much bribes");
          _bribesPerPriod[periods[i]][pool].push(rewarder);
      }
  }
```

All these details open the possibility any user to block the additional incentive for any of the pools available for voting **forever.**

## Impact

Blockage of the bribe rewarding mechanism.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L130-L144

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L226-L258

## Tool used

Manual Review

## Recommendation

Hard to give valid recommendation, but can think of making it permission required.

## [M-02] MlumStaking user can frontrun new reward tokens and harvest immediatelly

## Summary

All new rewards can be stolen by sandwiching rewards with 0 `lockDuration`.

## Vulnerability Detail

The way that users in `MlumStaking` receive rewards is first there should be some reward token transferred to the contract and the next caller will trigger `_updatePool` function, which will increase the `_accRewardsPerShare` based on the amount that has been sent and then divided by all the positions in the system.

```solidity
function _updatePool() internal {
        uint256 accRewardsPerShare = _accRewardsPerShare;
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        uint256 lastRewardBalance = _lastRewardBalance;

        // recompute accRewardsPerShare if not up to date
        if (lastRewardBalance == rewardBalance || _stakedSupply == 0) {
            return;
        }

        uint256 accruedReward = rewardBalance - lastRewardBalance;
        _accRewardsPerShare = accRewardsPerShare + ((accruedReward * (PRECISION_FACTOR)) / (_stakedSupplyWithMultiplier));

        _lastRewardBalance = rewardBalance;

        emit PoolUpdated(_currentBlockTimestamp(), accRewardsPerShare);
    }
```

The problem is that this `rewardToken` transfer can be sandwiched by anyone to harvest immediate rewards. It will happen since there is no minimum `lockDuration` and positions opened with 0 can be closed right after.

Impact is max when there is sequence of the following actions (note that due to the nature of IotaEVM, deployers will be forced to not submit 2 consecutive transactions for deploy and transferring reward tokens, since it can either fail or send the tokens to wrong address, depending on which transaction will be processed first, this gives the needed time for Alice to insert her tx between the 2):

1. 10e18 `rewardTokens` are sent, when there are no positions in the contract
2. Alice frontrun step 1 by `createPosition` with 1 wei and 0 `lockDuration`

Important variables:

- Alice’s `rewardDebt` = 0
- Alice’s `amountWithMultiplier` = 1
- `_stakedSupplyWithMultiplier` = 1
- `_accRewardsPerShare` = 0
1. Alice call `withdrawFromPosition` with 1 wei, and pool is updated:

```solidity
function _updatePool() internal {
        uint256 accRewardsPerShare = _accRewardsPerShare;
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        uint256 lastRewardBalance = _lastRewardBalance;

        // recompute accRewardsPerShare if not up to date
        if (lastRewardBalance == rewardBalance || _stakedSupply == 0) {
            return;
        }

        uint256 accruedReward = rewardBalance - lastRewardBalance;// 10e18
		    //@audit 0 + (10e18 * 1e12) / 1 = 10e30
        _accRewardsPerShare = accRewardsPerShare + ((accruedReward * (PRECISION_FACTOR)) / (_stakedSupplyWithMultiplier));

        _lastRewardBalance = rewardBalance;//10e18

        emit PoolUpdated(_currentBlockTimestamp(), accRewardsPerShare);
    }
```

1. In `_harvestPosition` the entire `rewardToken` balance is sent to Alice:

```solidity
function _harvestPosition(uint256 tokenId, address to) internal {
        StakingPosition storage position = _stakingPositions[tokenId];

        // compute position's pending rewards
        //@audit 1 * 10e30 / 10e12 - 0 = 10e18
        uint256 pending = position.amountWithMultiplier * _accRewardsPerShare / PRECISION_FACTOR - position.rewardDebt;

        // transfer rewards
        if (pending > 0) {
            // send rewards
            _safeRewardTransfer(to, pending);
        }
        emit HarvestPosition(tokenId, to, pending);
    }
```

1. All new `rewardToken` transfer can be immediately stolen.

Here is a POC demonstrating how Alice takes the entire balance of `rewardToken` **twice**. It can be placed in `MlumStaking.t.sol`:

```solidity
function testC1() public {
    _stakingToken.mint(ALICE, 10 ether);

    vm.startPrank(ALICE);
    _stakingToken.approve(address(_pool), 1 ether);
    _pool.createPosition(1 ether, 0);
    vm.stopPrank();

    IMlumStaking.StakingPosition memory pos = _pool.getStakingPosition(1);

    console.log("Rewards after create", _pool.pendingRewards(1));
    console.log("Rewards balance after create", _rewardToken.balanceOf(ALICE));
    console.log(pos.amountWithMultiplier);
    console.log(pos.rewardDebt);
    console.log(_pool._accRewardsPerShare());
    

    _rewardToken.mint(address(_pool), 100000e6);
    console.log("Rewards balance in staking", _rewardToken.balanceOf(address(_pool)));

    vm.prank(ALICE);
    _pool.withdrawFromPosition(1, 1 ether);

    console.log("Rewards balance after withdraw", _rewardToken.balanceOf(ALICE));
    console.log("RRewards balance in staking after", _rewardToken.balanceOf(address(_pool)));
    console.log(_pool._accRewardsPerShare());
    // =======================
    vm.startPrank(ALICE);
    _stakingToken.approve(address(_pool), 1 ether);
    _pool.createPosition(1 ether, 0);
    vm.stopPrank();
    console.log(_pool._accRewardsPerShare());

    _rewardToken.mint(address(_pool), 200e6);
    console.log("Rewards balance in staking", _rewardToken.balanceOf(address(_pool)));

    vm.prank(ALICE);
    _pool.withdrawFromPosition(2, 1 ether);
    console.log("Rewards balance after withdraw", _rewardToken.balanceOf(ALICE));
    console.log("RRewards balance in staking after", _rewardToken.balanceOf(address(_pool)));
    console.log(_pool._accRewardsPerShare());

}
```

Another possible scenario is when there are already some positions, everyone can sandwich the transferring of `rewardToken`, creating positions with 0 `lockDuration` and any satisfying amount of `stakedToken`, then backrun the transfer by withdraw and close the position taking the difference of previous `_accRewardsPerShare` and next one updated in `_updatePool`. This way no one will be incentivized to lock his tokens for any amount of time, when he can simply harvest the position immediately.

It order of events will look like that → `MlumStaking::createPosition` → `rewardToken::transfer(MlumStaking, 1000e18)` → `MlumStaking::withdrawFromPosition`.

## Impact

All the `rewardTokens` can be stolen from the first staker in `MlumStaking`

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354-L390

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L496-L502

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L674-L686

## Tool used

Manual Review

## Recommendation

Enforce some minimum stake duration

## [M-03] Anyone can call addToPosition

## Summary

Anyone can call `addToPosition()` for any tokenId in `MlumStaking.sol`, even if it should only be called by the owner or operators of the NFT.

## Vulnerability Detail

```solidity
function addToPosition(uint256 tokenId, uint256 amountToAdd) external override nonReentrant {
    _requireOnlyOperatorOrOwnerOf(tokenId); // AUDIT - can be called from anyone
    require(amountToAdd > 0, "0 amount"); // addToPosition: amount cannot be null

    _updatePool();
    address nftOwner = ERC721Upgradeable.ownerOf(tokenId);
    _harvestPosition(tokenId, nftOwner);

    StakingPosition storage position = _stakingPositions[tokenId];
    ...
    ...
}
```

`addToPosition()` uses `_requireOnlyOperatorOrOwnerOf()` which internally should check if the caller is the owner of the NFT or one of the contract operators, but in reality anyone will pass this check because it uses `ERC721Upgradeable._isAuthorized` and passes `msg.sender` for both `owner` and `spender` and this will always return true because `owner = spender`.

```solidity
function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
    // isApprovedOrOwner: caller has no rights on token
    require(ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId), "FORBIDDEN");
}
```

```solidity
function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
    return
        spender != address(0) &&
        (owner == spender || isApprovedForAll(owner, spender) || _getApproved(tokenId) == spender);
}
```

## Impact

Because of this, anyone can call `addToPosition` on any `tokenId` and grief when nftOwner wants to destroy his NFT (with full withdraw) by just frontrun him with `addToPosition()` for his NFT and deposit 1 wei.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L140-L143

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L398

## Tool used

Manual Review

## Recommendation

Change `_requireOnlyOperatorOrOwnerOf` to pass real owner of `tokenId` to `_isAuthorized()` and include operator checks as there are none now.

## [M-04] MlumStaking::addToPosition doesn’t consider fee-on-transfer tokens, unfairly calculating the avgDuration

## Summary

Tokens with fee-on-transfer mechanisms will unfairly increase the lock duration of positions in `MlumStaking::addToPosition`

## Vulnerability Detail

The team mentioned that all weird ERC20 tokens are in scope, this is applicable also for `MlumStaking`, as we can see there is a helper function to handle fee-on-transfer tokens:

```solidity
function _transferSupportingFeeOnTransfer(IERC20 token, address user, uint256 amount)
      internal
      returns (uint256 receivedAmount)
  {
      uint256 previousBalance = token.balanceOf(address(this));
      token.safeTransferFrom(user, address(this), amount);
      return token.balanceOf(address(this)) - previousBalance;
  }
```

The problem is in the `addToPosition` function and the fact that it uses the raw `amountToAdd` to calculate the new average duration of the lock, instead of the actual tokens deposited as it is being done in `createPosition`. In case token with high fee is being used as a `stakedToken`, duration will unfairly increased vs. highly decreased actual rewards, after fee has been taken from the transferred amount.

```solidity
function addToPosition(uint256 tokenId, uint256 amountToAdd) external override nonReentrant {
      _requireOnlyOperatorOrOwnerOf(tokenId);
      require(amountToAdd > 0, "0 amount"); // addToPosition: amount cannot be null

      _updatePool();
      address nftOwner = ERC721Upgradeable.ownerOf(tokenId);
      _harvestPosition(tokenId, nftOwner);

      StakingPosition storage position = _stakingPositions[tokenId];

      // we calculate the avg lock time:
      // lock_duration = (remainin_lock_time * staked_amount + amount_to_add * inital_lock_duration) / (staked_amount + amount_to_add)
      uint256 remainingLockTime = _remainingLockTime(position);
      uint256 avgDuration = (remainingLockTime * position.amount + amountToAdd * position.initialLockDuration)
          / (position.amount + amountToAdd);

      position.startLockTime = _currentBlockTimestamp();
      position.lockDuration = avgDuration;

      // lock multiplier stays the same
      position.lockMultiplier = getMultiplierByLockDuration(position.initialLockDuration);

      // handle tokens with transfer tax
      amountToAdd = _transferSupportingFeeOnTransfer(stakedToken, msg.sender, amountToAdd);

      // update position
      position.amount = position.amount + amountToAdd;
      _stakedSupply = _stakedSupply + amountToAdd;
      _updateBoostMultiplierInfoAndRewardDebt(position);

      emit AddToPosition(tokenId, msg.sender, amountToAdd);
  }
```

## Impact

Average durations extend is calculated before tax is taken, extending it unfairly, while the rewards that will be received for this period are calculated after deducting the fee, leading to worse ratio of time locked/rewards.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L397-L428

## Tool used

Manual Review

## Recommendation

Move this line: `amountToAdd = _transferSupportingFeeOnTransfer(stakedToken, msg.sender, amountToAdd);` before new lock time is recalculated.

## [M-05] harvestPositionsTo cannot be used as intended

## Summary

`harvestPositionsTo` cannot be used as intended and the **`to`** parameter is redundant due to incorrect check.

## Vulnerability Detail

MlumStaking has harvest function and tries to have 3 implementations.

`MlumStaking` has a `harvest` function and tries to have 3 implementations.

- Harvesting single NFT and passing the owner as recipient
- Harvesting single NFT and passing 3-party recipient
- Harvesting multiple NFTs and passing 3rd-party recipient

But the last one does not work as intended due to an unnecessary check.

```solidity
/**
 * @dev Harvest from multiple staking positions to "to" address
 *
 * Can only be called by lsNFT's owner or approved address
 */
function harvestPositionsTo(uint256[] calldata tokenIds, address to) external override nonReentrant {
    _updatePool();

    uint256 length = tokenIds.length;

    for (uint256 i = 0; i < length; ++i) {
        uint256 tokenId = tokenIds[i];
        _requireOnlyApprovedOrOwnerOf(tokenId);
        address tokenOwner = ERC721Upgradeable.ownerOf(tokenId);
        // if sender is the current owner, must also be the harvest dst address
        // if sender is approved, current owner must be a contract
        require(
            (msg.sender == tokenOwner && msg.sender == to), // legacy || tokenOwner.isContract()
            "FORBIDDEN"
        );

        _harvestPosition(tokenId, to);
        _updateBoostMultiplierInfoAndRewardDebt(_stakingPositions[tokenId]);
    }
}
```

This function is intended to be called by the `tokenOwner` or `approved` users as specified in the NatSpec and pass an arbitrary address **`to`** (who will receive the reward). But now only the owner can call and only the owner can be the recipient because of the `require` check**.**

**`require((msg.sender == tokenOwner && msg.sender == to), "FORBIDDEN");`**

## Impact

`to` address cannot differ from owner and approved users cannot call the function.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L470-L489

## Tool used

Manual Review

## Recommendation

Remove the require statement, this will allow approved users to call this and a different `to` address can be passed.