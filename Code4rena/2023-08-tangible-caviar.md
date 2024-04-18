Found: 4 high

<!--
# 115: https://github.com/code-423n4/2023-08-tangible-findings/issues/115
# 101: https://github.com/code-423n4/2023-08-tangible-findings/issues/101
# 99: https://github.com/code-423n4/2023-08-tangible-findings/issues/99
# 105: https://github.com/code-423n4/2023-08-tangible-findings/issues/105
 -->


# H1:
# Lines of code
https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarFeeManager.sol#L260

# Vulnerability details
## Impact
In function `CaviarFeeManager._distributeFees()`, there's this mistake by the dev for transfering Caviar token instead of USDC token. This's causing `CaviarFeeManager.notifyRewards()` and `CaviarFeeManager.distributeFees()` function behaving wrongly

```solidity
    function _distributeFees(uint256 _amount) internal {
        ...

        if(treasury != address(0)) {
            _swapUsdrToUsdc(_amountTngbl);
            uint256 _usdcBalance = IERC20(usdc).balanceOf(address(this));
            IERC20(caviar).safeTransfer(treasury, _usdcBalance); <@ NOTICE THIS
        }
    }```

## Tools Used
Manual Review
## Recommended Mitigation Steps
Change the line to `IERC20(usdc).safeTransfer(treasury, _usdcBalance);`





## Assessed type

Other
```




# H2:
# Lines of code
https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarCompounder.sol#L61-L65 https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarChef.sol#L184-L199

# Vulnerability details
## Impact
The function `CaviarCompounder.earn()` do external call `CaviarChef.deposit()`. `CaviarChef.deposit()` need CaviarCompounder's permission to spend its underlying token, but CaviarCompounder doesn't have any method to approve it, causing DOS forever to the function `earn()`

## Proof of Concept
The function `CaviarCompounder.earn()` do external call to `CaviarChef.deposit()`

```solidity
    function earn() public {
        caviarChef.harvest(address(this));
        uint256 _balance = balanceNotInPool();

        caviarChef.deposit(_balance, address(this)); <@ NOTICE THIS
    }
```

`CaviarChef.deposit()` require CaviarCompounder's approval to spend its underlying token, which CaviarCompounder never give permission and dont't have any method to give the permission

```solidity
    function deposit(uint256 amount, address to) public onlyWhitelisted nonReentrant {
        ...
        underlying.safeTransferFrom(msg.sender, address(this), amount); <@ NOTICE THIS

        emit Deposit(msg.sender, amount, to);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Approve underlying token to CaviarChef to spend before executing `CaviarChef.deposit()`

## Assessed type
ERC20




# H3:
# Lines of code
https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarCompounder.sol#L50-L59 https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarCompounder.sol#L68-L74

# Vulnerability details
## Impact
The function `deposit()` from `CaviarCompounder.sol` doesn't depositing any token. Hence, user can `deposit()` unlimited amount of caviar compounder token with no cost. Moreover, user can drain `CaviarCompounder.sol`'s Caviar token (if have any) by executing `withdraw()`

## Proof of Concept
Function `deposit()` doesn't require any token

```solidity
    function deposit(uint256 _amount) public nonReentrant {
        uint256 _pool = balance();
        uint256 _shares = 0;
        if (totalSupply() == 0) {
            _shares = _amount;
        } else {
            _shares = (_amount.mul(totalSupply())).div(_pool);
        }
        _mint(msg.sender, _shares);
    }
```

And user can drain the contract's Caviar token, by burning previously-minted Caviar Compound token

```solidity
    function withdraw(uint256 _shares) public nonReentrant {
        uint256 r = (balance().mul(_shares)).div(totalSupply());
        _burn(msg.sender, _shares);

        token.safeTransfer(msg.sender, r);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add this line: `token.safeTransferFrom(msg.sender, address(this), _amount);` in `deposit()`

## Assessed type
Token-Transfer


# H4:

# Lines of code
https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarFeeManager.sol#L38-L39 https://github.com/code-423n4/2023-08-tangible/blob/0938a42ea53266314e26d6a3540d532e39270ddc/contracts/CaviarFeeManager.sol#L42

# Vulnerability details
## Impact
Parameters `usdr`, `usdc`, `usdrToUsdcRoute` in `CaviarFeeManager.sol` is never set, causing DOS forever on those functions: `CaviarFeeManager.notifyRewards()`, `CaviarFeeManager.distributeFees()`

## Proof of Concept
Function `notifyRewards()` and `distributeFees()` use those parameters `usdr`, `usdc`, `usdrToUsdcRoute`

```solidity
    function notifyRewards() external keeper {
        uint256 _before = IERC20(usdr).balanceOf(address(this));<@ NOTICE THIS
        uint256 i;
        
        for (i = 0; i < tokens.length; i ++) {
            address _token = tokens[i];
            if (isToken[_token]) {
                _swapToUSDR(_token);
            }
        }
        uint256 _after = IERC20(usdr).balanceOf(address(this));<@ NOTICE THIS
        uint256 _notified = _after - _before;

        if (_notified >0) {
            _distributeFees(_notified);
        }
    }
```

```solidity
    function _distributeFees(uint256 _amount) internal {
        uint256 _amountStaking = _amount.mul(feeStaking).div(feeMultiplier);
        uint256 _amountTngbl = _amount.sub(_amountStaking);

        IERC20(usdr).safeApprove(caviarChef, 0); <@ NOTICE THIS
        IERC20(usdr).safeApprove(caviarChef, _amountStaking); <@ NOTICE THIS
        ICaviarChef(caviarChef).seedRewards(_amountStaking);

        if(treasury != address(0)) {
            _swapUsdrToUsdc(_amountTngbl);
            uint256 _usdcBalance = IERC20(usdc).balanceOf(address(this)); <@ NOTICE THIS
            IERC20(caviar).safeTransfer(treasury, _usdcBalance);
        }
    }

    function distributeFees(uint256 _amount) external restricted {
        _distributeFees(_amount);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add some functions to set those variables

## Assessed type
Other

