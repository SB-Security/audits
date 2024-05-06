![Logo](https://i.imgur.com/hxRaWjB.jpeg)

# **🛠️ Analysis -** WiseLending

## Overview

Wise Lending is a protocol that serves as a DeFi liquidity market with multiple sources of yield integrated. The yield strategies approach makes the protocol unique by removing the complexity for a normal user to manually manage his assets. In a trust-minimized approach protocol gives the freedom to its users to decide how they want to operate within the protocol, they can choose from pure lending/borrowing or one of the many configurations provided by the development team: 

- No APY, pure lending/borrowing
- APY from Wise + APY from Aave
- APY from Wise + APY from Pendle
- APY from Wise + APY from Pendle + APY from Aave

As simple as that.

**Key contracts of WiseLending for this audit are:**

- `WiseLending`: A main contract used for all actions in the protocol like lending, borrowing, repaying, and liquidations. It also manages the following important features for the protocol - borrow rates, LASA algorithm, users’s positions, and configuring pool token markets.
- `PositionNFT`: ERC721 contract which takes care of minting and reserving all the positions across the WiseLending, it makes possible for positions to be transferred on third-party markets in a P2P.
- `PendlePowerFarm`: Contract serving as an entry point for WiseLending to Pendle Finance, with the help of Balancer’s flash loans, users can open and close leveraged positions with Pendle LP tokens.
- `PendlePowerFarmToken`: ERC20 contract representing the Pendle LP tokens of a particular Pendle market, also serves as a powerFarm yield reward distributor.
- `PendlePowerFarmController`: Contract that manages both Pendle LP and Pendle market tokens, a person with `master` role can vote and claim Pendle rewards from it.
- `FeeManager`: Contract managing bad debt in the system, also organizes fee distribution and rewards claiming accumulated from borrowers paying interest.

## **System Overview**

We can observe 6 main parts in the WiseLending system:

- `WiseLending` - Consists of many contracts inheriting each other, each one serving a different purpose, storage modification, pool configuration, core lending, borrowing, repaying and liquidation logic, share price calculation, and scaling algorithm.
- `DerivativesOracles & OraclesHub` - The Entire WiseLending system relies on oracles as all calculations are performed in ETH value, `OraclesHub` wraps the logic of all `DerivativesOracles` and exposes a generic interface, exposing only the price-deriving functions.
- `FeeManager` - Entity used to receive fees from borrow interest accrued and manage bad debt that is split amongst all the participants in the system.
- `PowerFarms` - Set of contracts, allowing full interaction with `Pendle` and `WiseLending`, serving as an entry point for swapping Standardised Yield tokens for Pendle LPs and opening `WETH` borrow positions, with the help of Balancer flash loans.
- `WrapperHub` - Set of contracts, allowing full interaction with `Aave` yield and `WiseLending`, serving as an atomic way to provide liquidity to `Aave` and receive additional yield from depositing the `ATokens` in Wise and vice versa.
- `WiseSecurity` - Helper contracts, used for preparing the Curve pools, setting liquidation configuration, calculating the health state of positions, and additional safe checks to not push the system into a bad state.

## Approach Taken in Evaluating WiseLending

| Stage | Action | Details | Information |
| --- | --- | --- | --- |
| 1 | Compile and Run Test | [Installation](https://github.com/code-423n4/2024-02-wise-lending/tree/main) | A bit complex setup, caused by the big codebase. |
| 2 | Documentation review | [Documentation](https://wisesoft.gitbook.io/wise/) | Provides more general information showing Wise's advantages over competitors, not enough technical explanations. |
| 3 | Contest details | [Audit Details](https://github.com/code-423n4/2024-02-wise-lending/tree/main?tab=readme-ov-file#wise-lending-audit-details) | Lacking any contract explanation. Has an extensive known issues page, significantly lowering the attack surface. |
| 4 | Diagramming | Excalidraw | Drawing diagrams through the entire process of codebase evaluation. |
| 5 | Test Suits | [Tests](https://github.com/code-423n4/2024-02-wise-lending/tree/main/test) | Mix between Foundry and Hardhat makes the test review stage harder for the auditor. |
| 6 | Manual Code Review | [Scope](https://github.com/code-423n4/2024-02-wise-lending/tree/main?tab=readme-ov-file#files-in-scope) | Reviewed all the contracts in scope. |
| 7 | Special focus on Areas of Concern | [Areas of concern](https://github.com/code-423n4/2024-02-wise-lending/blob/main/bot-report.md) | Observing the known issues and bot report.  |

## Architecture Recommendations

### WiseLending:

Developers did a great job, extracting common logic into contracts and utilizing inheritance to make the understanding easy, despite the complexity and size of the protocol. Understandably attack surface is big, but parts that provide economic incentives for the attacker are well-thought-out and developed in a precise manner. Post-exploit modifications applied, completely remove the chance of it happening again. We can observe best development principles are being applied with some minor mistakes as not utilizing the power of Solidity and repetitive variables in some places. Separation of concerns and abstraction are applied in a great way, `external` functions contain reusable functions. 

Notes for improvement:

- Add events emissions in most of the important functions, currently, there are only for the solely deposits.
- When possible move the validations, such as zero values checks, to the external functions.
- `_removePositionData` can be simplified, removing the unnecessary if statements and optimizing the execution of the function.
- liquidation, where total pool assets are less than the asked from the liquidator, should be reconsidered and potentially reimplemented.

![Lasa Algorithm explanation](https://i.imgur.com/j6BtNAm.png)

Lasa Algorithm explanation

### DerivativesOracles & OraclesHub:

The decision to use both Uniswap TWAP and Chainlink oracle minimizes the oracle manipulations, however, this increases the gas costs of the transactions, adds additional complexity, and exposes potential denial-of-services. Wise lending heavily relies on the oracles for all types of actions across the entire system, but there is no room for decimal errors as everything is converted to ETH value and scaled to 18 decimals. 

`OracleHub` exposes 2 functions: `getTokensInETH` and `getTokensFromETH` for converting the assets to/from ETH price and scaling to proper decimals. Also, sequencer and oracle state checks are added which is important for L2 chains.

`DerivativeOracles` are modular with an easy way to be used by the `OracleHub`, this makes the upgradeability of oracles an easy task.

Notes for improvement:

- Consider adding functionality to replace oracles
- Most of the assets in the Wise Lending will be ETH derivatives such as `stETH`, consider extending the functionality to support different feeds such as `USD` ones.

### FeeManager:

Contracts with unique approach, serving two main purposes to handle bad debt and fee management. Bad debt incentive is implemented well with a possibility to attract users and cover the bad debt accrued, this greatly decreases the chance of Wise system to end up insolvent.

### PendlePowerFarm:

Set of farming strategy contracts are implemented well, with a good architecture, following the inheritance pattern, similar to WiseLending. The idea to reserve the `powerFarm` NFTs and not let the users use them in the other parts of the system is a big limitation, but indeed it removes the complexity of managing them and allowing malicious actions to be performed. Increasing the position leverage with flash loans will greatly attract users, especially in bullish markets. 

Notes for improvement:

- Hardcoded UniswapV3 pool fee will render most of the `PowerFarms` useless due to the price impact calculation after depositing into the `PendleToken` contract, caused by the low liquidity pools and the price change that will be caused when swapping.
- `PENDLE_ROUTER.addLiquiditySingleSy` should be able to set `minLpOut` parameter instead of hardcoding 0.
- When `Balancer` starts taking fees on flash loans, there will be failed transactions due to the slippage caused from the 4 swaps done in `_logicOpenPosition` and inability to borrow exact same amount and cover the flash loan + fees.
```solidity
    function _logicOpenPosition(
          bool _isAave,
          uint256 _nftId,
          uint256 _depositAmount,
          uint256 _totalDebtBalancer,
          uint256 _allowedSpread
      ) internal {
          uint256 reverseAllowedSpread = 2 * PRECISION_FACTOR_E18 - _allowedSpread;
    
          if (block.chainid == ARB_CHAIN_ID) {
              _depositAmount = _getTokensUniV3(
                  _depositAmount,
                  _getEthInTokens(ENTRY_ASSET, _depositAmount) * reverseAllowedSpread / PRECISION_FACTOR_E18,
                  WETH_ADDRESS,
                  ENTRY_ASSET
              );
          } //@audit first swap
    
          uint256 syReceived = PENDLE_SY.deposit({
              _receiver: address(this),
              _tokenIn: ENTRY_ASSET,
              _amountTokenToDeposit: _depositAmount,
              _minSharesOut: PENDLE_SY.previewDeposit(ENTRY_ASSET, _depositAmount)
          }); //@audit second swap
    
          (, uint256 netPtFromSwap,,,,) =
              PENDLE_ROUTER_STATIC.addLiquiditySingleSyStatic(address(PENDLE_MARKET), syReceived);
    
          (uint256 netLpOut,) = PENDLE_ROUTER.addLiquiditySingleSy({
              _receiver: address(this),
              _market: address(PENDLE_MARKET),
              _netSyIn: syReceived,
              _minLpOut: 0,
              _guessPtReceivedFromSy: ApproxParams({
                  guessMin: netPtFromSwap - 100,
                  guessMax: netPtFromSwap + 100,
                  guessOffchain: 0,
                  maxIteration: 50,
                  eps: 1e15
              })
          });//@audit third swap
    
          uint256 ethValueBefore = _getTokensInETH(ENTRY_ASSET, _depositAmount);
    
          (uint256 receivedShares,) = IPendleChild(PENDLE_CHILD).depositExactAmount(netLpOut);//@audit fourth swap
    
          uint256 ethValueAfter = _getTokensInETH(PENDLE_CHILD, receivedShares) * _allowedSpread / PRECISION_FACTOR_E18;
    
          if (ethValueAfter < ethValueBefore) {
              revert TooMuchValueLost();
          }
    
          WISE_LENDING.depositExactAmount(_nftId, PENDLE_CHILD, receivedShares);
    
          _borrowExactAmount(_isAave, _nftId, _totalDebtBalancer);
    
          if (_checkDebtRatio(_nftId) == false) {
              revert DebtRatioTooHigh();
          }
    
          _safeTransfer(WETH_ADDRESS, BALANCER_ADDRESS, _totalDebtBalancer);
      }
```
    

### PendlePowerFarmController:

Pendle LP rewards are tracked properly and the incentive to provide Pendle tokens is a good way to have more APY. Both `Controller` and `PowerFarmToken` contracts are at a mutual benefit and made it possible for Wise users to increase their rewards. 

Notes for improvement:

- `syncSupply` modifier of `PendlePowerFarmToken` can be refactored
    - sharePrice can be calculated only by the `underlyingLpAssetsCurrent` , `previewDistribution` will return 0 after 1st call.
    - `updateRewards` should call `redeemRewards` in order to not have discrepancies with the rewards in `Pendle`.

```solidity
modifier syncSupply() {
    _triggerIndexUpdate();
    _overWriteCheck();
    _syncSupply();
    _updateRewards();
    _setLastInteraction();
    _increaseCardinalityNext();
    uint256 sharePriceBefore = _getSharePrice();
    _;
    _validateSharePriceGrowth(_validateSharePrice(sharePriceBefore));
}
```

- state variables holding the same addresses should be merged into single ones and reused on all places, such as `PENDLE_MARKET` and `UNDERLYING_PENDLE_MARKET`, `PENDLE_CONTROLLER` and `PENDLE_POWER_FARM_CONTROLLER`
- most of the functions in `PendlePowerFarmToken` will cost huge amount of gas to be called because of the `syncSupply` making ~25 external calls.
- `_getSharePrice` can use only `underlyingLpAssetsCurrent`, without calling `previewUnderlyingLpAssets`, because it will return 0 in a subsequent calls.
- `lockPendle` misses chain check, as locking in `vePENDLE` can happen only on Ethereum.

### WrapperHub:

This wrapper contracts utilizing `Aave` , reduce the complexity of a user’s side to first receive `ATokens` then deposit them in Wise in order to earn additional APY. They make the process straightforward, containing all the needed logic to ensure seamless lending and borrowing of `ATokens`. Nothing significant to be improved there, only events should be renamed. Approvals are done on configuring in `setAaveTokenAddress`, removing the need of additional Aave pool approvals on supply.

### WiseSecurity:

These contracts are implemented well, with big set of functions, used all across the codebase, proper access control is applied. Both `WiseSecurity` and `WiseSecurityHelper` contain the right set of functions, also separated by access modifiers.

Notes for improvement:

- `getLiveDebtRatio` should not revert on blacklisted tokens, because liquidations are possible for them intentionally

## Codebase Quality Analysis

We consider the quality of `WiseLending` codebase as a bit overegineered, but on other hand developers managed to keep the overall complexity medium.

All the contracts are modular, each one serving its own purpose, there are contracts such as `WiseSecurity` and `PositionNFTs` that all others reuse. They are implemented well with a low likelihood denial-of-service to be caused, blocking the entire application. 

Approach taken for the lending and borrowing part of the system is good, it handles the bad debt realization different from the other protocols.

Single entity controls the entire protocol which exposes risk if it gets compromised or lost.

Tests are extensive but without any structure and proper order, making them extremely hard to be understood and auditor to grasp the flows of the actions in the system, additionally using two testing frameworks (hardhat and foundry), increases the complexity and should be avoided.

| Codebase Quality Categories | Comments |
| --- | --- |
| Code Maintainability and Reliability | The WiseLending contracts demonstrate good maintainability through modular structure, consistent naming, and efficient use of libraries. It also prioritizes reliability by handling errors, and conducting security checks. However, for enhanced reliability, consider professional security audits and documentation improvements. Regular updates are crucial in the evolving space. |
| Code Comments | While comments provided high-level context for certain constructs, lower-level logic flows and variables were not fully explained within the code itself. Following best practices like NatSpec could strengthen self-documentation. As an auditor without additional context, it was challenging to analyze code sections without external reference docs. To further understand the implementation intent for those complex parts, referencing supplemental documentation was necessary. |
| Documentation | Mostly non-technical documentation is provided on the WiseLending website and a decent amount of known issues are on the contest page, it would be more beneficial for both sponsors and auditors to have more technical documentation, explaining the scenarios in the system. |
| Code Structure and Formatting | The codebase contracts are well-structured and formatted. but some order of functions do not follow the Solidity Style Guide According to the Solidity Style Guide, functions should be grouped according to their visibility and order: constructor, receive, fallback, external, public, internal, and private. Within a grouping, place the view and pure functions last. |

## Test analysis

The overall test coverage is high, most of them are not really unit tests but more of a component testing. Development team should put additional effort to convert all the Hardhat tests to Foundry, as it will make subsequent audits more easier for both parties. Additionally tests, should be even more modular, testing each function in isolation. 

Fuzz & Invariant testing will suit this type of protocol, so that type of service should be considered, even 5% improvement is worth it in the world of smart contracts. 

Using both mocks and forks speaks well for the tests as they are put into real environment. After the exploit, there are separate files focusing on critical parts of the code, which is also good. 

On the other hand, we faced difficulties executing Hardhat tests, as most of them either were no compiling or reverting because of the wrong asserts in all files.

## **Systemic & Centralization Risks**

As stated in the protocol’s documentation - decentralization is a key element of the ecosystem and they are working towards it. 

`WiseLending` consist of small amount of roles, with one that controls everything across the contracts - `master`.

Leaving everything to governance will be the best possible approach but due to the nature of the protocol the chosen approach finds the sweet point.

However, having single entity responsible for everything can expose certain centralization risks, as it can be compromised or even behave maliciously ([Wonderland rugpull](https://www.reddit.com/r/CryptoCurrency/comments/16jtdi0/how_the_wonderland_project_crashed_almost_100/)), but observing the contracts we can see that there is no functions that can make this easily achievable, without going unnoticed by users.

`Master` user can set two other roles inside the `WiseLending`:

- Beneficials in `FeeManager`
- SecurityWorkers in `WiseSecurity`

`securityWorkers` can put the pool markets in a bad state if they decide, since they can freely pause the protocol.

 Protocol admins should consider giving some of these roles to the governance in the future, when test period passes and everything is settled and working seamlessly.

All important functions have proper access controls minimising the third party systemic and centralization risks.

## Team Findings

| Severity | Findings |
| --- | --- |
| High | 0 |
| Medium | 5 |
| Low/NC | 7 |

Most of the findings were identified in the `WiseLending` and `PendlePowerFarm`.

The ones with a significant impact are:

- `PowerFarm` enters will fail when Balancer starts taking fees on flash loans.
- Small positions without incentive for liquidations will lead to insolvency of the entire pool token market.
- All transactions in `PendlePowerFarmToken` will consume big amount of gas, possibly resulting in a DOS for the contract when many `rewardTokens` are presented in `PendleMarket`.
- Liquidator will brick the entire pool market, also stealing from the subsequent depositors.

## New insights and learning from this audit

Learned about:

- [Pendle](https://docs.pendle.finance/Home)
- [Balancer flash loan mechanism](https://docs.balancer.fi/reference/contracts/flash-loans.html)
- [LASA scaling algorithm](https://github.com/wise-foundation/liquidnfts-audit-scope/blob/master/LASA-Paper.pdf) used in WiseLending
- [Aave](https://docs.aave.com/developers/getting-started/readme)

## Conclusion

In general, `WiseLending` protocol presents a well-designed but a bit overengineered architecture. It exhibits, such as modularity, entity separation, proven integrations with third-party protocols and we believe the team has done a good job regarding the code. However, there are areas for improvement, including tests, code documentation and extensive technical documentation. Systemic and Centralization risks do not pose any significant impact to the system. 

Main code recommendations include simplifying redundant functions, if-statements rework, and extracting redundant state variables, holding same addresses.

It is also highly recommended that the team continues to invest in security measures such as mitigation reviews, audits, and bug bounty programs to maintain the security and reliability of the project.



### Time spent:
70 hours