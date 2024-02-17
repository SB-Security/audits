## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-erc20-token-that-is-not-18-decimals-is-not-supported) | ERC20 token that is not 18 decimals is not supported | High |
| [M-01](#m-01-chainlinklatestrounddata-price-not-checked) | Chainlink.latestRoundData price not checked | Medium |
| [M-02](#m-02-token-price-in-usd-will-be-wrong-when-the-tokens-usd-price-feed-is-decimals--8) | Token price in USD will be wrong when the token’s USD price feed is `decimals != 8` | Medium |
| [L-01](#l-01-zero-address-check-for-tokens) | Zero address check for tokens | Low |

# [H-01] ERC20 token that is not 18 decimals is not supported

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L324-L332

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L340-L348

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L71-L71

## Summary
Protocol is hard coded to ERC20 tokens with 18 decimals, which leads to wrong calculations and can produce a loss of funds when using tokens with differences in decimals.

## Vulnerability Details
For example, USDC has only 6 decimals, and if mix it with other tokens that have more decimals (ex: 18, 24), it will return the wrong calculation and always will assume the token with the large amount of decimals for the calculations which will break the precision and a lot of the functionality.

```solidity
// @audit test with more that one token collateral deposited, when deposited tokens have different decimals
function testIfHealthFactorIsWith18Decimals() public {
    vm.startPrank(user);    
    ERC20Mock(weth).mint(user, 10e24);
    ERC20Mock(weth).approve(address(dsce), amountCollateral);
    dsce.depositCollateralAndMintDsc(weth, amountCollateral, amountToMint);

    ERC20Mock(wbtc).approve(address(dsce), 10e10);
    dsce.depositCollateral(wbtc, 10e10);
    vm.stopPrank();

    int256 ethUsdUpdatedPrice = 18e8; // 1 ETH = $18
    
    // 180000000000000180000000000 - Total Collateral in USD
    // 900,000,000,000,000,900,000,000 Health factor for 
    // WETH with 24 decimals 100 deposited and WBTC with 10 decimals 100 deposited and 100 DSC minted


    MockV3Aggregator(ethUsdPriceFeed).updateAnswer(ethUsdUpdatedPrice);

    uint256 userHealthFactor = dsce.getHealthFactor(user);

    assert(userHealthFactor < 1e18);
}
```

https://prnt.sc/elsi2gbyWqH9

## Instances

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L324-L332
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L340-L348
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L71-L71

## Impact
High, the whole protocol precision calculations are hardcoded to 18 decimals.

## Tools Used
Manual Review

## Recommendations
Add support for ERC20 tokens with different decimals that `18` by checking `decimals()` when make calculations. If protocol is only for ERC20 tokens with 18 decimals, checks should be added in the constructor.

# [M-01] Chainlink.latestRoundData price not checked

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/libraries/OracleLib.sol#L26C9-L27C41

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L340-L348

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L361-L367

## Summary
AggregatorV3Interface.latestRoundData function returns the price of a specific asset. The price comes as a signed integer and has to be checked because there are possible scenarios where Chainlink oracle can return zero or even worse negative answer.

Source: 
https://youtu.be/a5G6k6NFsCg?t=134

## Vulnerability Details
The function `priceFeed.staleCheckLatestRoundData()` can return a negative price(int256) which after that is cast to uint256().
Currently, this oracle function is used in 2 places: `getUsdValue` and `getTokenAmountFromUsd`. 
The problem is most likely to occur in the second function, especially on this specific line where we calculate the token amount from USD for the passed collateral. 

In case of answer equal to 0, <ins>division by zero</ins> will occur on this line:

```solidity
return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);
```

If the returned `answer` is lower than 0, there will be <ins>silent underflow</ins>. Let's assume that the oracle's `answer` is -1, after cast we will receive this number:
```solidity
uint256(-1) = 115792089237316195423570985008687907853269984665640564039457584007913129639935
```
## Impact
Liquidations will be blocked because `getTokenAmountFromUsd` will always <ins>revert</ins> when additional precision is applied (1e10) so it will be more than type(uint256).max.

If the price is 0 it will lead to division by 0 in getTokenAmountFromUsd and getUsdValue functions.
## Tools Used
Manual
## Recommendations
Check if the price is greater than 0. Consider using OpenZeppelin’s SafeCast library to prevent unexpected overflows when casting from uint256.

# [M-02] Token price in USD will be wrong when the token’s USD price feed is `decimals != 8`

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L347

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L366

## Summary
The response from the Chainlink Oracle price feed always assumes 8 decimals, However, there are certain tokens where USD feed has different decimals.

## Vulnerability Details
In the current implementation, the price conversion is hard coded to work when price feed decimals are 8.
`((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION`

However, there are tokens with USD price feed's decimals != 8 (e.g.: `AMPL / USD` feed decimals = 18)

(AMPL / USD) Price Feed  -  https://etherscan.io/address/0xe20CA8D7546932360e37E9D72c1a47334af57706

## Instances

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L347

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L366

## Impact
When the price feed with `decimals != 8` is set, can lead to incorrect conversion and potentially draining all of the funds.

## Tools Used
Manual Review / Foundry

## Recommendations
Add a check for price feed decimals in the OracleLib library to prevent the precision loss, or add a check in the constructor and only allow price feeds with 8 decimals.

```solidity
constructor(address[] memory tokenAddresses, address[] memory priceFeedAddresses, address dscAddress) {
    // USD Price Feeds
    if (tokenAddresses.length != priceFeedAddresses.length) {
        revert DSCEngine__TokenAddressesAndPriceFeedAddressesMustBeSameLength();
    }
    // For example ETH / USD, BTC / USD, MKR / USD, etc
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
    +   if (AggregatorV3Interface(priceFeedAddresses[i]).decimals() != 8) {
    +       revert DSCEngine__PriceFeedDecimals();
    +   }
        s_priceFeeds[tokenAddresses[i]] = priceFeedAddresses[i];
        s_collateralTokens.push(tokenAddresses[i]);
    }
    i_dsc = DecentralizedStableCoin(dscAddress);
}
```

# [L-01] Zero address check for tokens

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol#L112-L123

## Summary
In the current implementation the token and price feed addresses aren’t checked for zero address upon initialization, there is a modifier which catch scenarios when price feed with zero address will be passed, but not for token addresses.

## Vulnerability Details
When deploy the `DSCEngine.sol`, if pass token with address(0) and working price feed address, the deployment will be successful, but the user experience is going to fall when using the protocol, due to EVM Revert.

```solidity
// Deploying the protocol localy with token address(0)

return NetworkConfig({
        wethUsdPriceFeed: address(ethUsdPriceFeed),
        wbtcUsdPriceFeed: address(btcUsdPriceFeed),
        weth: address(0),
        wbtc: address(wbtcMock),
        deployerKey: DEFAULT_ANVIL_KEY
    });
```

## Impact
It will make freshly deployed DSCEngine unusable and the protocol deployer will have to redeploy everything.

## Tools Used
Manual, Foundry

## Recommendations
Add a check in the constructor

```solidity
constructor(address[] memory tokenAddresses, address[] memory priceFeedAddresses, address dscAddress) {
    // USD Price Feeds
    if (tokenAddresses.length != priceFeedAddresses.length) {
        revert DSCEngine__TokenAddressesAndPriceFeedAddressesMustBeSameLength();
    }
    // For example ETH / USD, BTC / USD, MKR / USD, etc
    for (uint256 i = 0; i < tokenAddresses.length; i++) {
    +   if (tokenAddresses[i] == address(0) || priceFeedAddresses[i] == address(0)) {
    +       revert DSCEngine__TokenAddressZero();
    +   }
        s_priceFeeds[tokenAddresses[i]] = priceFeedAddresses[i];
        s_collateralTokens.push(tokenAddresses[i]);
    }
    i_dsc = DecentralizedStableCoin(dscAddress);
}
```

```solidity
function getTokenAmountFromUsd(address token, uint256 usdAmountInWei) public view returns (uint256) {
    // price of ETH (token)
    // $/ETH ETH ??
    // $2000 / ETH. $1000 = 0.5 ETH
+   if (token == address(0)) {
+       revert DSCEngine__NotAllowedToken();
+   }
    AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
    (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();

    // ($10e3 * 1e18) / ($2000e8 * 1e10)
    return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);
}
```

```solidity
function getUsdValue(address token, uint256 amount) public view returns (uint256) {
+   if (token == address(0)) {
+       revert DSCEngine__NotAllowedToken();
+   }

    AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
    (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
    // 1 ETH = $1000
    // The returned value from CL will be 1000 * 1e8
    return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
}
```