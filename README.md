# Solidity Gas Optimizations

Thanks to [PatrickAlphaC](https://github.com/PatrickAlphaC) for amazing courses

--------------

## Lock Pragma Version
I know this is not completely related to gas optimization. But, let's start from the beginning.

Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

--------------

## uint256 is cheaper than uint8
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

contract Counter {
    uint8 private counter;
}
```
| Variable | Gas Cost |
|---|---|
| counter | 54394 | 





