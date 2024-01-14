## Overview

[ENS](https://github.com/code-423n4/2023-10-ens) is a decentralized naming system for wallets websites, and & more. However, contracts that are currently being audited are related to the governance part of the organization. The unique approach allows token holder to delegate his votes to multiple users with the help of proxy contracts.

## Understanding the Ecosystem

ENS is built around various standards and modules. But the currently reviewed contract is more related to: 
 

- `ERC20Votes`: A smart contract standard that includes two main components, `ERC20` and `Votes`, which provides generic way for the delegator to give his voting tokens to another user.
- `ERC1155`:  Multi-Token standard which offers a comprehensive range of essential features, the most important ones for this ecosystem are `_mintBatch`, `_burnBatch`, which are used to respectively **mint** tokens for a user who is delegating his voting power to delegate or **burn** the same tokens which in the context of the protocol is used to reimburse the owner of delegated tokens, and `balanceOf` whose function, in this case, is to track the balance of delegate and the tokens he is being delegated of.
- `Ownable`: AccessControl standard which sets the owner of `ERC20MultiDelegate` contract and exposes the important `onlyOwner` modifier used in the `setUri` function.

## Codebase Quality Analysis

- The ENS contract is well-structured and follows good practices for smart contract development. It has a single point of entry `delegateMulti()` function which is used either to transfer votes between delegates, reimburse the delegator, or delegator to give his votes to multiple delegates.
- The contract lacks up-to-date documentation to describe the use case and to explain how things are going to work. The only way for the user or auditor in this case to understand the codebase is to dive deep into the code and tests and gain his own assumptions of how things are working, possibly wrong, which can affect the quality of the audit.
- There are a good amount of tests and good code coverage, but they can be split into helper functions, for example:

```solidity
const delegates = [deployer, alice, bob, charlie];
const amounts = delegates.map(() =>
  delegatorTokenAmount.div(delegates.length)
);
```

```solidity
await multiDelegate.delegateMulti([], delegates, amounts);
```

These 2 code snippets are used in multiple places and extracting them into functions will reduce the complexity of the tests. Also new tests can be added in order to validate the edge cases of the `ERC1155` given the fact that some of the functions are used differently than for what they are intended. 

- There are unused but deployed contracts in the tests - `Registry` and `Resolver` which don’t give any particular value to the tests and therefore increase the complexity and test execution time.

## Architecture Recommendations

The architecture of `ERC20MultiDelegate` is well-designed with unique approach which allows utilizing the `ERC1155` and proxies by handling delegation to more than one people at the same time. However there are small improvements that could be made:

- Improve the input validation by adding address(0), amount == 0 checks. There is also lacking check if **source address** is equal to the **target address** in `processDelegation` function, by adding them contract will be more gas efficient and unnecessary `ERC20ProxyDelegator` deployments will be avoided.
- Implement functions to retrieve accidentally send tokens to the `ERC20MultiDelegate` , now they are stuck inside forever.
- As discussed above at the #**Codebase quality analysis** point, tests can be modified as well.

## ****Centralization Risks****

- Aside from the `setUri` function which has **onlyOwner** modifier we can state there is little to none centralisation in the reviewed contracts, even on the contrary, with the whole idea of using a **proxy** and **ERC1155** together which allows users to delegate their votes to multiple people at the same time results increased decentralization of the whole **ENS governance.**

## Mechanism Review

The `ERC20MultiDelegate` contract is innovative as it uses `ERC1155` tokens to create a unique composite token structure involving delegators and delegate(proxies). This clever approach removes the need for traditional data structures like mappings and arrays to handle user-delegate connections. The idea that a single proxy can store votes from multiple users is intriguing, and the entire system operates on the basis of minted ERC1155 tokens.

However, as previously mentioned, the absence of a check to verify if the transferred amount is 0 could potentially allow users to spam multiple proxies. On the flip side, the mechanisms employed in the contract are indeed fascinating, providing valuable learning experiences for all involved.

## Systemic Risks

- Despite the areas of improvement mentioned earlier, main contract `ERC20MultiDelegate` has no any particular risks. Sponsor has shown some areas where problems might occur, but after a thorough investigation, we were unable to find anything critical, which indicates that the developers have done their job by avoiding unnecessary complexity and thereby shrinking the attack surface

## Codebase Analysis

- The ENS codebase is well-structured, but with lack of documentation and small and modular tests . However, there are areas where improvements could be made, particularly in terms of gas efficiency, documentation and tests.

## Recommendations

To improve the `ERC20MultiDelegate` and `ERC20ProxyDelegator` consider adding the changes that we recommended at the **#Architecture Recommendations** point:

- Add zero address checks
- Add check for amount which is being send through one of the `transferFrom` functions to be more than 0
- Add **admin only** function to be able to retrieve the accidentally transferred tokens to both contracts.

## ****Conclusion****

- The both contract (`ERC20MultiDelegate` and `ERC20ProxyDelegator`) are well-designed and their innovative approach by utilizing **Proxies and ERC1155** in order to allow votes to be delegated to multiple users at once, without any unnecessary complexity, will be great addition to the whole ENS ecosystem and for the ETH community itself.

### Time spent:
40 hours