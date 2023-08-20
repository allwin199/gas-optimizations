# Solidity Gas Optimizations

Thanks to [PatrickAlphaC](https://github.com/PatrickAlphaC) for amazing courses

--------------

## Lock Pragma Version
I know this is not completely related to gas optimization. But, let's start from the beginning.

Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

--------------

## uint256 is cheaper than uint8
### uint8
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint8 private counter;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 54406 | 

### uint256
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint256 private counter;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 54394 | 

--------------

## No need to initialize variables with default values
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint256 private counter = 0;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 54458 | 

In the previous example, `counter` was not initialized and resulting in less gas

--------------

## Visibility Matters
### Public
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint256 public counter;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 56206 | 

### Private
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint256 public counter;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 54394 | 

--------------

## Arrange storage variables
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    bool state;
    address owner;
    uint256 counter;
}
```
- bool is 1byte and address is 20bytes, so they can fit in a single slot(each slot is 32 bytes)  
Refer [here](https://medium.com/@bloqarl/solidity-gas-optimization-1-understanding-how-evm-works-can-save-you-gas-44c87011b295)

--------------

## Use Custom Errors instead of strings
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract CustomError {

    error MoreThanZero();

    function fund() public payable {
        if(msg.value <= 0){
            revert MoreThanZero();
        }
    }
}
```

--------------

## Use !=0 instead of <=0
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract CustomError {

    error MoreThanZero();

    function fund() public payable {
        if(msg.value != 0){
            revert MoreThanZero();
        }
    }
}
```
since it is uint, we can check like this. This is gas efficient.






