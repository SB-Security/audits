## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-consider-use-stethuds-oracle) | Consider use stETH/UDS oracle | High |
| [H-02](#h-02-first-depositor-will-get-twice-more-minted-token-for-the-same-amount-deposited) | First depositor will get twice more minted token for the same amount deposited | High |
| [H-03](#h-03-first-deposit-of-1-wei-will-block-further-rseth-minting) | First deposit of 1 Wei will block further rsETH minting | High |
| [L-01](#l-01-no-removesupportedasset-function-in-the-lrtconfig) | No removeSupportedAsset function in the LRTConfig | Low |
| [L-02](#l-02-cbeth-has-blacklisting) | cbETH has blacklisting | Low |
| [L-03](#l-03-deposit-can-be-blocked-due-to-steth-rebasing) | Deposit can be blocked due to stETH rebasing | Low |

# [H-01] Consider use stETH/UDS oracle

**Issue Description: The sponsor has confirmed their choice of Chainlink as an oracle to fetch prices.** Since all other LST price feeds are 18 decimal places, they will most likely use `stETH/ETH` price feeds. However, this feed has a long heartbeat and a 2% deviation threshold, which could lead to fund loss. The 24-hour heartbeat and 2% deviation threshold mean the price can move up to 2% or remain unchanged for 24 hours before triggering a price update. This could result in the on-chain price being significantly different from the true stETH price, leading to incorrectly calculated rsETH to mint.

**Recommendation:**

Consider using the `stETH/USD` oracle instead, as it offers a 1-hour heartbeat and a 1% deviation threshold. To accommodate this change, also need to adjust the decimal calculation in `LRTOracle::getAssetPrice` to multiply by `18 ** (token.decimal - pricefeed.decimal)` or a similar factor. This adjustment ensures that the price of all tokens is returned with the same decimal precision.

# [H-02] First depositor will get twice more minted token for the same amount deposited

## Impact

The initial depositor stands to gain an unfair amount of RSETH tokens compared to later depositors, as a result of the fixed exchange rate of **1 ether** when no RSETH supply exists (i.e., no minted tokens are available).

Consequently, the first deposit will receive ETH price of deposited token **(rETH price in term of ETH at the moment: 1,090,300,000,000,000,000)** in RSETH tokens.

## Proof of Concept

As mentioned earlier, the current price of rETH in terms of ETH is 1,090,300,000,000,000,000. 

Obtain this information from [0x536218f9E9Eb48863970252233c8F271f554C2d0](https://etherscan.io/address/0x536218f9E9Eb48863970252233c8F271f554C2d0)

![Chainlink-priceFeed](https://user-images.githubusercontent.com/84782275/283229965-ae2e52bc-b204-478e-9e4a-3c3e385a9b67.png)

![Answer](https://user-images.githubusercontent.com/84782275/283229971-445b250c-8c2f-4095-a942-3865d06862d8.png)

1. Alice deposits 1e18 tokens of any supported asset can be either rETH, cbETH, or stETH (will use rETH for the explanation).
2. The code calculates the RSETH token amount using the formula:

```solidity
src: src/LRTDeposit#L109
rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();
```

Considering it's the first deposit, `lrtOracle::getRSETHPrice` returns a hardcoded value of 1 ether (1e18):

```solidity
uint256 rsEthSupply = IRSETH(rsETHTokenAddress).totalSupply();
if (rsEthSupply == 0) {
    return 1 ether;
}
```

For a single `rETH` token, the contract will mint `the value of rETH taken from the chainlink at the moment` RSETH tokens for Alice.

1. Bob deposit 1e18 token.
2. In `depositAsset()`, it starts by adding the depositAmount and then calculates the amount of rsETH tokens to be minted.
3. `getRsETHAmountToMint()` will be called.

In this line, the amount on the left stays the same as the initial deposit, but the value from lrtOracle.getRSETHPrice() on the right side will change.

```solidity
rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();
//                   1e18 * 1,090,300,000,000,000,000 / ? (calculated in the next steps)
```

1. `getRSETHPrice()` will get the `rsEthSupply` which will be 1,090,300,000,000,000,000. It then calculates the totalETHInPool based on the two deposited 1e18 rETH tokens from Alice and Bob.
2. It will get 2e18 and will multiple it by 1,090,300,000,000,000,000 (price of rETH at the moment)

```solidity
return totalETHInPool / rsEthSupply; 
// (2e18 * 1,090,300,000,000,000,000) / 1,090,300,000,000,000,000
// = (2e18 * ~~1,090,300,000,000,000,000~~) / ~~1,090,300,000,000,000,000~~
// = 2e18
```

```solidity
rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();
//                   1e18 * 1,090,300,000,000,000,000 / 2e18
//                   = 545,150,000,000,000,000
```

1. In the end, Bob will deposit the same amount but will receive half as many `rsETH` tokens (545,150,000,000,000,000) compared to the Alice's initial deposit.
2. If a third person participates, they will also receive `545,150,000,000,000,000` rsETH tokens following the same pattern.

### Coded POC

Import `console` ,`StdUtils`,`MockToken` and `RSETH` at the top of the `LRTDepositPoolTest.t.sol` file.

```solidity
import { console } from "forge-std/Test.sol";
import { StdUtils } from "forge-std/StdUtils.sol";
import { MockToken } from "./LRTConfigTest.t.sol";
import { RSETH } from "src/RSETH.sol";
```

Place the PoC at the end of the file.

Can be run with: 

```
forge test --match-contract LRTDepositPoolInitialDeposit --match-test test_InitialDepositorGetTwiceMoreRSETHMinted -vvv
```

```solidity
contract LRTOracleMock_InitialDepositor {
    RSETH rseth;
    MockToken public rETH;
    LRTDepositPool public lrtDepositPool;

    constructor(RSETH rsEth, MockToken reth, LRTDepositPool LrtDepositPool) {
        rseth = rsEth;
        rETH = reth;
        lrtDepositPool = LrtDepositPool;
    }

    function getAssetPrice(address) public pure returns (uint256) {
        return 1_090_300_000_000_000_000; // Price of rETH at the moment of write
    }

    function getRSETHPrice() external returns (uint256) {
        uint256 rsETHTotalSupply = rseth.totalSupply();

        if (rsETHTotalSupply == 0) {
            return 1 ether;
        }

        return (rETH.balanceOf(address(lrtDepositPool)) * getAssetPrice(address(0))) / rsETHTotalSupply;
    }
}

contract LRTDepositPoolInitialDeposit is LRTDepositPoolTest {
    address public rETHAddress;
    LRTOracleMock_InitialDepositor public lrtOracle;

    function setUp() public override {
        super.setUp();

        // initialize LRTDepositPool
        lrtDepositPool.initialize(address(lrtConfig));

        rETHAddress = address(rETH);

        // add manager role within LRTConfig
        vm.startPrank(admin);
        lrtConfig.setContract(LRTConstants.LRT_ORACLE, address(new LRTOracleMock_InitialDepositor(rseth, rETH, lrtDepositPool)));

        lrtConfig.grantRole(LRTConstants.MANAGER, manager);
        vm.stopPrank();
    }

    function test_InitialDepositorGetTwiceMoreRSETHMinted() external {
        uint256 depositAmount = 1 ether;

        vm.startPrank(alice);

        rETH.approve(address(lrtDepositPool), depositAmount);
        lrtDepositPool.depositAsset(address(rETH), depositAmount);
        
        uint256 aliceRSETHBalance = rseth.balanceOf(alice);
        console.log("Alice rsETH Balance: ", aliceRSETHBalance);
        vm.stopPrank();

        vm.startPrank(bob);

        rETH.approve(address(lrtDepositPool), depositAmount);
        lrtDepositPool.depositAsset(address(rETH), depositAmount);
        
        uint256 bobRSETHBalance = rseth.balanceOf(bob);
        console.log("Bob rsETH Balance:    ", bobRSETHBalance);
    }
}
```

```
Logs:
  Alice rsETH Balance:  1090300000000000000
  Bob rsETH Balance:     545150000000000000
```

## Tools Used

Manual Review, Foundry

## Recommended Mitigation Steps

An effective solution to address this issue would be to ensure that the initial deposit in the DepositPool is made by the admin or one of the managers. Alternatively, a minimum deposit requirement of, for instance, 1 token (rETH, cbETH, or stETH) could be implemented. This way, all users will receive an equal amount of rsETH tokens when depositing the same amount of assets.

# [H-03] First deposit of 1 Wei will block further rsETH minting

## Impact

If the initial deposit in the DepositPool is 1 wei of any supported token (rETH, cbETH, or stETH), 1 wei of rsETH will be minted for the first depositor. However, subsequent `rsETH` minting will be prevented because the `rsethAmountToMint` will always round down to 0, resulting in users not receiving any tokens for their deposits. 

## Proof of Concept

In the situation where the initial deposit in the pool is `1 wei`, the first depositor will receive 1 wei of rsETH. Even if users deposit the remaining amount up to the deposit limit `(100,000 ether - 1 wei)`, they will still receive `0 rsETH` due to rounding issues.

> Note: Will use rETH as an asset and also consider its price in terms of ETH based on the Chainlink oracle 
- `rETH / ETH` at [0x536218f9E9Eb48863970252233c8F271f554C2d0](https://etherscan.io/address/0x536218f9E9Eb48863970252233c8F271f554C2d0) = 1,090,300,000,000,000,000 (at the moment of writing)
> 

![Chainlink](https://user-images.githubusercontent.com/84782275/283229965-ae2e52bc-b204-478e-9e4a-3c3e385a9b67.png)

Flow:

1. Alice deposit first in the DepositPool 1 wei of rETH.
2. 1 wei of rsETH will be minted for her.
3. Bob deposits the remaining rETH amount up to the deposit limit, which is `100,000 ether - 1 wei`.
4. Bob’s tokens will be deposited inside the `lrtDepositPool`, but he will not receive any rsETH tokens due to rounding issues.

### Coded POC

Import `console` ,`StdUtils`,`MockToken` and `RSETH` at the top of the `LRTDepositPoolTest.t.sol` file.

```solidity
import { console } from "forge-std/Test.sol";
import { StdUtils } from "forge-std/StdUtils.sol";
import { MockToken } from "./LRTConfigTest.t.sol";
import { RSETH } from "src/RSETH.sol";
```

Place the PoC at the end of the file.

Can be run with: 

```
forge test --match-contract LRTDepositPoolInitialDepositOf1Wei --match-test test_InitialDepositorOf1WeiWillResultInNoMoreRSERHToBeMinted -vvv
```

```solidity
contract LRTOracleMock_InitialDepositor {
    RSETH rseth;
    MockToken public rETH;
    LRTDepositPool public lrtDepositPool;

    constructor(RSETH rsEth, MockToken reth, LRTDepositPool LrtDepositPool) {
        rseth = rsEth;
        rETH = reth;
        lrtDepositPool = LrtDepositPool;
    }

    function getAssetPrice(address) public pure returns (uint256) {
        return 1_090_300_000_000_000_000;
    }

    function getRSETHPrice() external returns (uint256) {
        uint256 rsETHTotalSupply = rseth.totalSupply();

        if (rsETHTotalSupply == 0) {
            return 1 ether;
        }

        return (rETH.balanceOf(address(lrtDepositPool)) * getAssetPrice(address(0))) / rsETHTotalSupply;
    }
}

contract LRTDepositPoolInitialDepositOf1Wei is LRTDepositPoolTest {
    address public rETHAddress;
    LRTOracleMock_InitialDepositor public lrtOracle;

    function setUp() public override {
        super.setUp();

        // initialize LRTDepositPool
        lrtDepositPool.initialize(address(lrtConfig));

        rETHAddress = address(rETH);

        // add manager role within LRTConfig
        vm.startPrank(admin);
        lrtConfig.setContract(LRTConstants.LRT_ORACLE, address(new LRTOracleMock_InitialDepositor(rseth, rETH, lrtDepositPool)));

        lrtConfig.grantRole(LRTConstants.MANAGER, manager);
        vm.stopPrank();
    }

    function test_InitialDepositorOf1WeiWillResultInNoMoreRSERHToBeMinted() external {
        uint256 depositAmountUpToAssetLimit = 100_000 ether - 1;

        vm.startPrank(alice);

        // Depositing 1 wei
        rETH.approve(address(lrtDepositPool), 1);
        lrtDepositPool.depositAsset(address(rETH), 1);
        
        uint lrtDepositPoolRETHBalanceAfterAliceDeposit = rETH.balanceOf(address(lrtDepositPool));
        console.log("DepositPool rETH balance after Alice deposits 1 wei: ", lrtDepositPoolRETHBalanceAfterAliceDeposit);

        uint256 aliceRSETHBalance = rseth.balanceOf(alice);
        console.log("Alice rsETH Balance: ", aliceRSETHBalance);
        vm.stopPrank();

        vm.startPrank(bob);

        rETH.approve(address(lrtDepositPool), depositAmountUpToAssetLimit);
        lrtDepositPool.depositAsset(address(rETH), depositAmountUpToAssetLimit);

        uint lrtDepositPoolRETHBalanceAfterBobDepositRestOfTheDepositLimit = rETH.balanceOf(address(lrtDepositPool));
        console.log("DepositPool rETH balance after Bob deposits (100 000 ether - 1 wei) : ", lrtDepositPoolRETHBalanceAfterBobDepositRestOfTheDepositLimit);
        
        uint256 bobRSETHBalance = rseth.balanceOf(bob);
        console.log("Bob rsETH Balance:   ", bobRSETHBalance);
        vm.stopPrank();
    }
}
```

## Tools Used

Manual Review, Foundry

## Recommended Mitigation Steps

A potential solution would be to ensure that the admin or one of the managers makes the initial deposit in the DepositPool, or alternatively, set a minimum deposit requirement, perhaps 1 token (rETH, cbETH, or stETH). This approach can help address the issue of users receiving 0 `rsETH` due to rounding.

# [L-01] No removeSupportedAsset function in the LRTConfig

**Issue Description:** The contract provides a way to add new supported assets through the `addNewSupportedAsset()` function, but there is no corresponding function to remove supported assets.

Without the ability to remove supported assets, the contract may face challenges in adapting to changing circumstances, such as changes in the project's strategy, token ecosystem, or regulatory requirements.

**Recommendation:** Consider adding a function like **`removeSupportedAsset`** that allows the contract owner or another authorized role to remove an asset from the list of supported assets

# [L-02] cbETH has blacklisting

**Issue Description:** The blacklisting mechanism in the `cbETH` token introduces potential complications and risks across various stages of the transaction lifecycle, from initial deposits in the LRTDepositPool to activities within the NodeDelegator and Eigen Strategy. If the `LRTDepositPool` or any of the `NodeDelegator` is blacklisted, it implies that the associated tokens will become trapped in these contracts.

# [L-03] Deposit can be blocked due to stETH rebasing

## Impact

Each asset (LST) intended for deposit must adhere to a deposit limit of 100,000e18. It's important to note that `stETH`, being a rebasing token, has a variable balance that can fluctuate (balance could go up and down).

Due to the nature of stETH's balance changes, the `depositLimit` invariant may become inaccurate. Consequently, deposits may be temporarily blocked until the `stETH` balance falls below the deposit limit or until the manager updates it.

## Proof of Concept

stETH has two rebasing scenarios, let's delve into each of them.

### **Positive Rebase:**

If a user deposits stETH and it's close to reaching the deposit limit. On the next deposit, he will follow these steps:

1. He will call `getAssetCurrentLimit()` to determine the allowable deposit amount before reaching the asset deposit limit.
2. Assuming the function returns the correct value and there has been no rebase since the initial deposit.
3. The user will obtain `X` (the amount of tokens they can deposit).
4. Will call `depositAsset(stETH, X)`.
5. However, just before the user's deposit, a rebase occurs, causing the total value of previously deposited assets to exceed the deposit limit.
6. As a consequence, **`getAssetCurrentLimit()`** will revert due to **`getTotalAssetDeposits()`** returning a value higher than 100,000e18.
7. His deposit, as well as any further calls to `getAssetCurrentLimit()`, will be blocked.

Here is a coded PoC:

Import `console` and `StdUtils` at the top of the `LRTDepositPoolTest.t.sol` file.

```solidity
import { console } from "forge-std/Test.sol";
import { StdUtils } from "forge-std/StdUtils.sol";
```

Place the PoC in the `LRTDepositPoolDepositAsset` contract.

Can be run with: 

```
forge test --match-contract LRTDepositPoolDepositAsset --match-test test_DepositRevertAfterStETHRebase -vvv
```

```solidity
function test_DepositRevertAfterStETHRebase() external {
    uint256 depositAmount = 99_999 ether;

    vm.startPrank(alice);

    stETH.approve(address(lrtDepositPool), 100_000 ether);
    // Will deposit 99,999 - close to the limit
    lrtDepositPool.depositAsset(address(stETH), depositAmount);

    uint256 aliceAvailableAmountToDeposit = lrtDepositPool.getAssetCurrentLimit(address(stETH));
    console.log("Alice avaiable amount to deposit before rebase occur: ", aliceAvailableAmountToDeposit); // 1 ether

    // deposit pool balance of stETH before rebase
    uint256 depositPoolBalanceBefore = stETH.balanceOf(address(lrtDepositPool));
    console.log("Deposit pool stETH balace before rebase: ", depositPoolBalanceBefore);
    // Simulate stETH rebase
    deal(address(stETH), address(lrtDepositPool), depositAmount + 10 ether);
    // deposit pool balance of stETH after rebase
    uint256 depositPoolBalanceAfter = stETH.balanceOf(address(lrtDepositPool));
    console.log("Deposit pool stETH balace after rebase: ", depositPoolBalanceAfter);
    
    // Deposit will revert
    vm.expectRevert();
    lrtDepositPool.depositAsset(address(stETH), aliceAvailableAmountToDeposit);

    // further calls to getAssetCurrentLimit() will revert also
    vm.expectRevert();
    lrtDepositPool.getAssetCurrentLimit(address(stETH));
}
```

```
Logs:
Alice avaiable amount to deposit before rebase occur:  1000000000000000000
Deposit pool stETH balace before rebase:  99999000000000000000000
Deposit pool stETH balace after rebase:  100009000000000000000000
```

### Negative Rebase

In a rare situation, the amount of `stETH` can decrease, called a negative rebase.

There is a scenario where the asset limit is reached, but a negative rebase will make some extra space and whoever sees this will be able to mint additional `rsETH`.

1. Numerous users have made deposits, reaching the `stETH` deposit limit.
2. User back-run the `stETH` negative rebase allowing them to deposit an additional amount until they reach the limit again.
3. As a result, extra `rsETH` is minted for this specific user.

## Tools Used

Manual Review

## Recommended Mitigation Steps

It's challenging to provide a coded recommendation due to the inherent nature of the stETH token and its rebasing mechanism, which cannot be halted. However, I offer the following suggestion:

Consider using **`wstETH`** as it simplifies integration with DeFi and eliminates the rebasing functionality. This can provide a more stable and predictable environment for your application.