# ****🛠️ Analysis - KelpDAO - rsETH****

---

## Overview

---

KelpDAO aims to launch a new and advanced Liquid Restaked Token called rsETH. rsETH represents a consolidated, re-staked token that can be generated using LSTs accepted as collateral within the EigenLayer ecosystem. The idea is to make restaking and diving into DeFi super easy, playing well with other DeFi stuff. KelpDAO is here to fix tricky reward systems and those pesky high gas fees, making a big splash in the web3 scene. To break it down, the Kelp protocol takes in deposits, mainly in the form of LSTs recognized as EigenLayer collateral. In return, users receive $rsETH – KelpDAO's distinctive reward token, symbolizing each individual's deposited assets.

**The key contracts of KelpDAO - $rsETH protocol for this Audit are**:

- LRTConfig: This upgradeable contract plays a pivotal role in storing the configuration settings for the Kelp product. Additionally, it serves as a repository for the addresses of other contracts within the Kelp product, establishing a centralized hub for managing various aspects of the system's functionality.
- LRTDepositPool: Operating as an upgradeable contract, LRTDepositPool functions as a receiving point for funds deposited by users into the Kelp product. Subsequently, these funds are channeled to NodeDelegator contracts, facilitating the efficient movement of resources within the Kelp ecosystem.
- NodeDelegator: These contracts serve as intermediaries, receiving funds from the LRTDepositPool and strategically delegating them to the EigenLayer strategy. The allocated funds are then utilized to provide liquidity within the EigenLayer protocol. Notably, NodeDelegator is designed as an upgradeable contract, allowing for adaptability and future enhancements.
- LRTOracle: Functioning as another upgradeable contract, LRTOracle is tasked with the responsibility of fetching accurate price data for Liquid Staking Tokens from external oracle services. This ensures that the Kelp product has real-time and reliable information to make informed decisions within its operations.
- RSETH: As an upgradeable ERC20 token contract, RSETH serves as a receipt token for deposits made into the Kelp product. This specialized token represents the user's deposit within the system and is designed with upgradability to align with any future modifications or improvements to the Kelp ecosystem.

## **System Overview**

---

### Scope

- LRTConfig - A core upgradeable contract that contains the state variables used across the entire protocol. It is responsible for storing and modifying them.
    
    Can also change the following core variables:
    
    - Adding/removing assets (LST tokens, such as stETH, cbETH, and rETH)
    - Limits for depositing into EigenLayer
    - The **Strategy** used in the NodeDelegators
    - External contracts used across the Kelp DAO
- LRTDepositPool - An upgradeable contract, that serves as an entry point for the user, it receives the funds deposited by a user and with the help of **LRTManager** sends them to the NodeDelegators.
    
    Additional operations are available:
    
    - Querying the deposits for the entire protocol (total assets deposited, deposit limits, assets deposited in EigenLayer and in all the NodeDelegators)
    - Depositing assets to the Kelp DAO products
    - Calculating the amount of RSETH tokens to be minted to the user
    - Minting RSETH tokens to depositors
    - Transferring funds to NodeDelegators
    - Pausing/Upausing the important functionalities
- NodeDelegator - Mediator contracts that are designed to receive funds from LRTDepositPool and delegate them to EigenLayer and vice-versa.
    
    Possible operations:
    
    - Approvals to EigenLayer’s StrategyManager contract
    - Depositing into the desired EigenLayer strategy
    - Transferring tokens back to the LRTDepositPool
    - Fetches the balance of the assets deposited into EigenLayer through this NodeDelegator contract
    - Pausing/Unpausing
- LRTOracle - an upgradeable contract that is responsible for fetching the price of Liquid Staking Tokens tokens from oracle services.
    
     Price queries available:
    
    - Fetching the exchange rates of LST tokens in ETH
    - Calculating the price of RSETH tokens based on all the available asset
    - Adding/removing price oracles used as PriceFetchers
    - Pausing/unpausing functions

### Privileged Roles

Privileged roles used across Kelp DAO:

- LRTManager

```solidity
modifier onlyLRTManager() {
        if (!IAccessControl(address(lrtConfig)).hasRole(LRTConstants.MANAGER, msg.sender)) {
            revert ILRTConfig.CallerNotLRTConfigManager();
        }
        _;
    }
```

- LRTAdmin

```bash

    modifier onlyLRTAdmin() {
        bytes32 DEFAULT_ADMIN_ROLE = 0x00;
        if (!IAccessControl(address(lrtConfig)).hasRole(DEFAULT_ADMIN_ROLE, msg.sender)) {
            revert ILRTConfig.CallerNotLRTConfigAdmin();
        }
        _;
    }
```

### Trusted Roles

```solidity
MINTER_ROLE - given to the LRTDepositPool at deployment
BURNER_ROLE - TBC, will be given to the contract which will be used to withdraw from EigenLayer
MANAGER(LRTManager): 
	- can pause all the contracts that implement PausableUpgradeable from OpenZeppelin
	- can approve assets that have to be deposited into EigenLayer
	- can deposit assets into the EigenLayer strategy 
	- can transfer back assets to the LRTDepositPool contract
ADMIN(LRTAdmin):
	- can add new NodeDelegatorContracts to LRTDepositPool mapping
	- can update the max NodeDelegators count allowed in the LRTDepositPool
	- can unpause all the contracts that implement PausableUpgradeable from OpenZeppelin
User - Can deposit LST tokens into the protocol and see the assets available 
```

## Approach Taken-in Evaluating The Kelp DAO Protocol

---

| Number | Stage | Details | Information |
| --- | --- | --- | --- |
| 1 | Compile and Run Test | Installation | Clearly written and easy to setup, with the additional script to generate test coverage |
| 2 | Architecture Review | [Kelp DAO](https://blog.kelpdao.xyz/) | Provides a basic LST and architectural overview without emphasizing on the technical side |
| 3 | Graphical Analysis | Graphical Analysis with [Solidity-Metrics](https://github.com/ConsenSys/solidity-metrics) | A visual view has been made to dominate the general structure of the codes of the project. |
| 4 | Slither Analysis | [Slither Report](https://github.com/crytic/slither) | The project does not currently have a slither result. We ran local slither and no major issue were identified. |
| 5 | Test Suits | [Tests](https://github.com/code-423n4/2023-11-kelp#tests) | In this section, the scope and content of the tests of the project are analyzed. |
| 6 | Manual Code Review | [Scope](https://github.com/code-423n4/2023-11-kelp#scope) | Reviewed all the contracts in scope |
| 7 | Infographic | [Article](https://medium.com/@0xmilenov/the-story-of-alice-and-bob-in-kelp-dao-closer-look-at-the-flow-inside-the-smart-contracts-of-the-83a56bab79bc) | Reviewed medium article written about the contracts in scope |
| 8 | Special focus on Areas of  Concern | [Areas of Concern](https://github.com/code-423n4/2023-11-kelp#known-issues) | Observing the known issues and bot report |

## Architecture Recommendations

---

**Comments**: While the codebase contains comments, improving the quality of comments and documentation can enhance code readability. Consider using more explanatory comments to clarify the purpose and functionality of each section of the code. This will make it easier for other developers and auditor to understand and maintain the code.

**Documentation**: In overall, protocol team relies more on the high-level overview of the codebase, instead of describing the technical details and make the auditability of the protocol easier. 

For example this is the only more technical article by the team: 
https://blog.kelpdao.xyz/decoding-the-tech-behind-rseth-409ecddcb421

Which explains only a part of the components of which the project is build, such as what is **rsETH and his usages.** Additional documentation regarding the whole flow will be beneficial for all the parties involved in the protocol. ****

**Formal Verification**: Consider a professional formal verification of the contract to identify and mitigate any security risks.

**External dependencies:** Consider implementing reliable oracle, instead of `[ChainlinkPriceOracle](https://github.com/code-423n4/2023-11-kelp/blob/main/src/oracles/ChainlinkPriceOracle.sol)` which contains stale function `latestAnswer` and doesn’t comply with any of the suggested security measures.

**Variable naming:** Some of the variable are not well named, which can cause the confusion for the person who is reading them, there are some examples:

- `asset_idx` is not self explanatory, can be replaced with **index, assetIndex**
- `nodeDelegatorQueue` is not appropriate and sounds more like a queue to us, consider simplifying it to `nodeDelegators`

![image](https://user-images.githubusercontent.com/58271928/283230855-ead19442-939c-4b86-9470-d3f025175c94.png)

## Codebase Quality Analysis

---

We consider the quality of the Kelp DAO as simple and robust, but incomplete, since there are functions which have to be added, such as `withdraw` both from EigenLayer and the protocol itself, because now there is no way for user to withdraw his deposited assets and the only way to unstake from EigenLayer is to rely to the DelegationManager of the Strategy to call `withdrawSharesAsTokens` function located in their StrategyManager contract.

| Codebase Quality Categories | Comments | Score (1-10) |
| --- | --- | --- |
| Code Maintainability and Reliability | The codebase demonstrates moderate maintainability with well-structured functions but lacks proper function ordering based on their visibility. The approach that the development team has taken is different from the preferred one in the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html#order-of-functions). In Kelp DAO, functions are grouped by functionality instead of visibility. While this grouping may be better for readability, the pros of grouping by visibility are better from a development point of view. Adherence to standard conventions and practices contributes to overall code quality. Some variable names are not quite understandable and can lead to confusion for the developer and auditor. The usage of the UtilLib library to abstract the zero address check significantly reduces lines of code and improves readability. Exposing functions to directly check balances in all contracts improves UX and provides an easy way for potential depositors to verify required values. | 7 |
| Code Comments | The contracts have comments explaining the purpose and functionality of different parts, making it easier for developers to understand and maintain the code. Comments describe methods, variables, and the overall structure of the contract. For example, the code comments declare imported interfaces and contracts and provide descriptions of the contract's title and purpose. | 9 |
| Documentation | There is no extensive documentation to explain the architecture and functionality of the protocol, only a brief explanation of the contracts in scope. Documentation for the integration with EigenLayer is lacking, and there is no information on what will happen when the user deposits his funds. Diagrams must be drawn to represent how end-to-end flow is executed, given the fact this protocol will be used as a fund aggregator and deposit them into another protocol. Consider using flows from the Architecture Overview point. | 3 |
| Code Structure and Formatting | The codebase contracts are well-structured and formatted, utilizing many of the OpenZeppelin contracts which are well-tested and documented. This lowers the attack surface and makes future development and auditability easier. | 10 |

## Test analysis

---

The audit scope of the contracts to be audited is ~97% and it should be aimed to be 100%.
A good amount of testing should be applied to the script files, as they contain critical parameters that have to be verified. 

## **Systemic & Centralization Risks**

The analysis provided highlights several significant systemic and centralization risks present in the Kelp DAO protocol. These risks encompass concentration risk in all the contracts in scope and centralization risks arising from the existence of an `LRTManager` and `LRTOwner` roles in all the contracts. However, the documentation lacks clarity on whether this address represents an externally owned account (EOA) or a contract, warranting the need for clarification. Additionally, the absence of deployment scripts tests could also pose risks to the protocol’s security since they have to be manually verified now. Regarding the tokens used, they are not quite similar and errors can occur due to rebasing of some of them.

Here's an analysis of potential systemic and centralization risks in the provided contracts:

### Systemic Risks:

1. No deployment scripts tests, which can introduce irreversible damage for the protocol caused by wrong addresses.
2. A compromise of the admin, gatekeeper or minter roles would allow unauthorized minting/redeeming, threatening stability.
3. stETH is a rebase token and can cause DOS in certain times when volatility of the token is high and due to rebase deposit limit can be hit, which will disallow everyone to deposit their funds in the EigenLayer.
4. First depositor can brick the entire protocol if deposits 1 wei causing the other depositors to just donate tokens into the protocol, due to rounding issue in the `getRsETHAmountToMint` function.
5. No way to remove supported asset after they are deprecated by EigenLayer, because of lacking remove function in the LRTConfig.

### Centralization Risks:

1. High centralization as most of the core functions are restricted only to a `LRTManager` and `LRTAdmin` contracts.

**Properly managing these risks and implementing best practices in security and decentralization will contribute to the sustainability and long-term success of the Kelp DAO protocol.**

## New insights and learning from this audit

---

Learned about: 

- [Idea behind Kelp DAO](https://blog.kelpdao.xyz/)
- [TransparentUpgradeable contracts](https://medium.com/coinmonks/transparent-upgradable-smart-contracts-a-guide-with-code-explanation-465a486ca1df)
- [Liquid Staking Tokens](https://liquidcollective.io/liquid-staking/)
- [Yield-Bearing Tokens](https://academy.stakedao.org/what-are-yield-bearing-tokens/)

## Conclusion

---

In general, the Kelp DAO project exhibits an standard and well-developed architecture we believe the team has done a good job regarding the code, but the identified risks need to be addressed, and measures should be implemented to protect the protocol from potential malicious use cases. Additionally, it is recommended to improve the documentation and comments in the code to enhance understanding and collaboration among developers. It is also highly recommended that the team continues to invest in security measures such as mitigation reviews, audits, and bug bounty programs to maintain the security and reliability of the project.


### Time spent:
30 hours