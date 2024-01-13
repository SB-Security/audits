## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-lender-logic-is-based-only-for-erc20-tokens-with-18-decimals) | Lender logic is based only for ERC20 tokens with 18 decimals | High |
| [H-02](#h-02-cei-is-not-followed-which-makes-functions-vulnerable-to-reentrancy) | Lender contract can be drained by re-entrancy in `repay` | High |
| [H-03](#h-02-cei-is-not-followed-which-makes-functions-vulnerable-to-reentrancy) | Lender contract can be drained by re-entrancy in `setPool` | High |
| [H-04](#h-03-lack-of-slippage-protection-when-using-iswaprouter) | Lack of slippage protection when using ISwapRouter | High |
| [L-01](#l-01-missing-zero-address-check) | Missing zero address check | Low |
| [L-02](#l-02-missing-check-if-loans-and-pools-arrays-are-equal-length) | Missing check if loans and pools arrays are equal length | Low |
| [L-03](#l-03-not-emitting-events-for-important-state-changes) | Not emitting events for important state changes | Low |

# [H-01] Lender logic is based only for ERC20 tokens with 18 decimals

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L246

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L384

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L618

## Summary
The `Lender.sol` contract is hardcoded to work only with 18 decimals ERC20 tokens.

## Vulnerability Details
In the current implementation loanRation in borrow(), giveLoan(), and refinance() functions are hardcoded to 18 decimals calculations.

```solidity
src\Lender.sol

591: function refinance(Refinance[] calldata refinances) public {
    for (uint256 i = 0; i < refinances.length; i++) {
		...
		...
		// If the debt is with 10 decimals and collateral is with 24
		// 50e10 debt - 120e24 collateral = the loanRation will be 41,666 no decimals and the whole calculation will be a total mess
		if (pool.poolBalance < debt) revert LoanTooLarge();
        if (debt < pool.minLoanSize) revert LoanTooSmall();
        uint256 loanRatio = (debt * 10 ** 18) / collateral;
        if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();
		...
		...
		}
}
```

## Impact
Loss of precision, mixing tokens with different decimals can lead to loss of funds.

## Tools Used
Manual

## Recommendations
Consider restricting what tokens can be used if want to leave these calculations, or use ERC20.decimals() to prevent precision loss.

# [H-02] CEI is not followed which makes functions vulnerable to reentrancy

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L130-L176

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L232-L287

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L292-L345

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L355-L432

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L465-L534

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L548-L586

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L591-L710

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42

https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53-L58

## Summary
Most of the functions in `Lender.sol` and `Staking.sol` don’t follow the check-effect-interactions pattern and don’t implement any type of reentrancy guard.

## Vulnerability Details
In the following functions: 

- Lender.sol
    - `setPool()`
    - `borrow()`
    - `repay()`
    - `giveLoan()`
    - `buyLoan()`
    - `seizeLoan()`
    - `refinance()`
- Staking.sol
    - `deposit()`
    - `claim()`

All of these functions are open for reentrancy, they interact with external and most likely malicious ERC20s before changing the internal state of the contract.

## Poc (Proof-of-Concept)
Will give some examples of the most vulnerable functions which are easy to understand.
```solidity
// Here the user can call `claim` function to claim his reward for the swap.
// updateFor() will be called which is not in our interest because it's not updating anything that will prevent reentrancy
// After that we transfer WETH to msg.sender amount, and if use any malicious ERC20s can produce reentrancy
// The claimable mapping is reset after that, as well as the overall balance of WETH in the contract
// These state changes need to be made before all external calls to prevent any logic which will run these external calls

src\Staking.sol
53: function claim() external {
    updateFor(msg.sender);
    WETH.transfer(msg.sender, claimable[msg.sender]);
    claimable[msg.sender] = 0;
    balance = WETH.balanceOf(address(this));
}
```

```solidity
// If anyone creates a pool with malicious ERC20s and then you want to borrow from this pool.
// You will call borrow() the function will go until the transfers come, if loanToken or collateralTokens are not Safe can recall this function again,
// which will lead to the feeReceiver getting the fees again, msg.sender may get more than the asked debt or this contract can drain user collateral, there are a lot of scenarios.

src\Lender.sol
232: function borrow(Borrow[] calldata borrows) public {
    for (uint256 i = 0; i < borrows.length; i++) {
        bytes32 poolId = borrows[i].poolId;
        uint256 debt = borrows[i].debt;
        uint256 collateral = borrows[i].collateral;
        // get the pool info
        Pool memory pool = pools[poolId];
        // make sure the pool exists
        if (pool.lender == address(0)) revert PoolConfig();
        // validate the loan
        if (debt < pool.minLoanSize) revert LoanTooSmall();
        if (debt > pool.poolBalance) revert LoanTooLarge();
        if (collateral == 0) revert ZeroCollateral();
        // make sure the user isn't borrowing too much
        uint256 loanRatio = (debt * 10 ** 18) / collateral;
        if (loanRatio > pool.maxLoanRatio) revert RatioTooHigh();
        // create the loan
        Loan memory loan = Loan({
            lender: pool.lender,
            borrower: msg.sender,
            loanToken: pool.loanToken,
            collateralToken: pool.collateralToken,
            debt: debt,
            collateral: collateral,
            interestRate: pool.interestRate,
            startTimestamp: block.timestamp,
            auctionStartTimestamp: type(uint256).max,
            auctionLength: pool.auctionLength
        });
        // update the pool balance
        _updatePoolBalance(poolId, pools[poolId].poolBalance - debt);
        pools[poolId].outstandingLoans += debt;
        // calculate the fees
        uint256 fees = (debt * borrowerFee) / 10000;
        // transfer fees
        IERC20(loan.loanToken).transfer(feeReceiver, fees);
        // transfer the loan tokens from the pool to the borrower
        IERC20(loan.loanToken).transfer(msg.sender, debt - fees);
        // transfer the collateral tokens from the borrower to the contract
        IERC20(loan.collateralToken).transferFrom(msg.sender, address(this), collateral);
        loans.push(loan);
        emit Borrowed(
            msg.sender, pool.lender, loans.length - 1, debt, collateral, pool.interestRate, block.timestamp
        );
    }
}
```

```solidity
// Here you want to repay your loan, so call this function.
// It will get the loan, calculate the interest, updatePoolBalance and subtract your debt.
// then the loanToken will be transferred from you to the contract as well as interest for the protocol
//Let's assume that the collateralToken is malicious and on transferFrom will get from this contract to you and recall this 
// and because the loan deletion is made after the external calls everything can be executed again until drained the whole pool. 

src\Lender.sol
292: function repay(uint256[] calldata loanIds) public {
    for (uint256 i = 0; i < loanIds.length; i++) {
        uint256 loanId = loanIds[i];
        // get the loan info
        Loan memory loan = loans[loanId];
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(loan);

        bytes32 poolId = getPoolId(loan.lender, loan.loanToken, loan.collateralToken);

        // update the pool balance
        _updatePoolBalance(poolId, pools[poolId].poolBalance + loan.debt + lenderInterest);
        pools[poolId].outstandingLoans -= loan.debt;

        // transfer the loan tokens from the borrower to the pool
        IERC20(loan.loanToken).transferFrom(msg.sender, address(this), loan.debt + lenderInterest);
        // transfer the protocol fee to the fee receiver
        IERC20(loan.loanToken).transferFrom(msg.sender, feeReceiver, protocolInterest);
        // transfer the collateral tokens from the contract to the borrower
        IERC20(loan.collateralToken).transfer(loan.borrower, loan.collateral);
        emit Repaid(
            msg.sender, loan.lender, loanId, loan.debt, loan.collateral, loan.interestRate, loan.startTimestamp
        );
        // delete the loan
        delete loans[loanId];
    }
}
```

## Impact
Draining both lenders’ and borrowers’ funds, as well as `Staking.sol` contract, will make the entire protocol unusable.

## Tools Used
Manual

## Recommendations
Most important is to order your functions by following the CEI pattern and consider using ReentrancyGuard from OpenZeppelin. All state changes need to be made before interacting with external contracts.

Example of followed CEI on affected functions.
```solidity
src\Lender.sol
292: function repay(uint256[] calldata loanIds) public {
    for (uint256 i = 0; i < loanIds.length; i++) {
        uint256 loanId = loanIds[i];
        // get the loan info
        Loan memory loan = loans[loanId];
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(loan);

        bytes32 poolId = getPoolId(loan.lender, loan.loanToken, loan.collateralToken);

        // update the pool balance
        _updatePoolBalance(poolId, pools[poolId].poolBalance + loan.debt + lenderInterest);
        pools[poolId].outstandingLoans -= loan.debt;
				
+		// delete the loan
+       delete loans[loanId];				

        // transfer the loan tokens from the borrower to the pool
        IERC20(loan.loanToken).transferFrom(msg.sender, address(this), loan.debt + lenderInterest);
        // transfer the protocol fee to the fee receiver
        IERC20(loan.loanToken).transferFrom(msg.sender, feeReceiver, protocolInterest);
        // transfer the collateral tokens from the contract to the borrower
        IERC20(loan.collateralToken).transfer(loan.borrower, loan.collateral);
        emit Repaid(
            msg.sender, loan.lender, loanId, loan.debt, loan.collateral, loan.interestRate, loan.startTimestamp
        );
-       // delete the loan
-       delete loans[loanId];
    }
}
```
```solidity
src\Staking.sol
53: function claim() external {
    updateFor(msg.sender);
+		uint256 reward = claimable[msg.sender];
-   WETH.transfer(msg.sender, claimable[msg.sender]);
    claimable[msg.sender] = 0;
    balance = WETH.balanceOf(address(this));

+   WETH.transfer(msg.sender, claimable[msg.sender]);
}
```
```solidity
src\Staking.sol
38: function deposit(uint _amount) external {
-   TKN.transferFrom(msg.sender, address(this), _amount);
    updateFor(msg.sender);
    balances[msg.sender] += _amount;
+   TKN.transferFrom(msg.sender, address(this), _amount);
}
```

# [H-03] Lack of slippage protection when using ISwapRouter

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26-L44

## Summary
When using ISwapRouter from UniswapV3, _staking will accept the swap even when the amount returned is 0, because it is hardcoded in the ExactInputSingleParams.
## Vulnerability Details
In Uniswap’s documentation, there an article on how to execute swap transaction and especially what value to set the vulnerable `amountOutMin` param:
• amountOutMinimum: we are setting to zero, but this is a significant risk in production. As a result this can lead to loss of staking funds due to sandwich attacks.

```solidity

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: _profits,
            tokenOut: WETH,
            fee: 3000,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amount,
            amountOutMinimum: 0, //@audit there amountOutMinimum is specified to 0, this means trade will be accepted even 
                                 //if amount returned is 0
            sqrtPriceLimitX96: 0
        });
```
## Impact
Let’s suppose that address *_staking* wants to trade **any** ERC20 token to WETH. He executes the transaction and it goes to the **mempool**. A bot sniffs out the transaction and Front-Runs the **_staking** by purchasing WETH before the large trade is approved. This purchase raises the price of asset-WETH for the **_staking** trader and increases the slippage (Expected price increase or decrease in price based on the volume to be traded and the available liquidity). 
## Tools Used
Manual
## Recommendations
As supposed in the Uniswap docs `_amountOunMin` param's value should be calculated using their SDK or an on-chain price oracle - this helps protect against getting an unusually bad price for a trade due to a front-running sandwich or another type of price manipulation.

Guide:
https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps

# [L-01] Missing zero address check

## Summary
The addresses passed in constructors and functions aren’t checked for zero address.

## Vulnerability Details
Missed zero address checks in the following code:
```solidity
src\Staking.sol
31: constructor(address _token, address _weth) Ownable(msg.sender) {
    TKN = IERC20(_token);
    WETH = IERC20(_weth);
}
```
```solidity
src\Fees.sol
19: constructor(address _weth, address _staking) {
    WETH = _weth;
    staking = _staking;
}
```
```solidity
src\Lender.sol
100: function setFeeReceiver(address _feeReceiver) external onlyOwner {
  feeReceiver = _feeReceiver;
}
```

## Impact
`Fees.sol` and `Staking.sol` constructors don’t check for zero addresses, which will break the contract, becoming useless, and redeploying will be required to have a fully functional contract. As a result, these functions will become useless which will make the owner unable to receive his funds and lock fees from modifications.

## Tools Used
Manual

## Recommendations
Add checks for zero address.
```solidity
src\Staking.sol
31: constructor(address _token, address _weth) Ownable(msg.sender) {
    if (_token == address(0) || _weth == address(0)) {
        revert NotValidToken();
    }
    TKN = IERC20(_token);
    WETH = IERC20(_weth);
}
```

```solidity
src\Fees.sol
19: constructor(address _weth, address _staking) {
    if (_weth == address(0) || _staking == address(0)) {
        revert NotValidToken();
    }
    WETH = _weth;
    staking = _staking;
}
```
```solidity

src\Lender.sol
100: function setFeeReceiver(address _feeReceiver) external onlyOwner {
    if (_feeReceiver == address(0)) {
        revert FeeReceiverCannotBeZeroAddress();
    }
    feeReceiver = _feeReceiver;
}
```

# [L-02] Missing check if loans and pools arrays are equal length

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355-L432

## Summary
Function giveLoan() doesn’t check whether loanIds and poolIds arrays have the same length.
## Vulnerability Details
There is no check if loanIds.length and poolIds.length are equal. To match the selling loan to the new pool.
## Impact
If loanIds array is longer the function will revert when checking if pool.loanToken != loan.loanToken because all of the pool members will be default and pool.loanToken will be equal to zero address. 
## Tools Used
Manual
## Recommendations
Add a check in the start of the function to confirm that loanIds and poolIds have equal length.
```solidity
function giveLoan(uint256[] calldata loanIds, bytes32[] calldata poolIds) external {
		if (loanIds.length != poolIds.length) revert MismatchedArrayLengths();
    for (uint256 i = 0; i < loanIds.length; i++) {
        ...
}
```

# [L-03] Not emitting events for important state changes

## Summary
When changing state variables events are not emitted.
## Vulnerability Details
src\Lender.sol

`setLenderFee` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L84-L87](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L84C1-L87C6)

`setBorrowerFee` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L92-L95](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L92C1-L95C6)

`setFeeReceiver` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L100-L102](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol#L100C1-L102C6)

src\Staking.sol

`deposit` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38-L42](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L38C1-L42C6)

`withdraw` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46-L50](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L46C1-L50C6)

`claim` function

[https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53-L58](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol#L53C1-L58C6)
## Impact
The end user may be confused about state changes
## Tools Used
Manual
## Recommendations
Emit proper events so that off-chain monitoring can be implemented.

For `setLenderFee` function emit event `LenderFeeUpdated(uint indexed fee)`

For `setBorrowerFee` function emit event `BorrowerFeeUpdated(uint indexed fee)`

For `setFeeReceiver` function emit event `ReceiverFeeUpdated(uint indexed fee)`

For `deposit` function emit event `Deposited(address indexed from, uint256 amount)`

For `withdraw` function emit event `Withdrawn(address indexed from, uint256 amount)`

For `claim` function emit event `Claimed(address indexed from, uint256 amount)`