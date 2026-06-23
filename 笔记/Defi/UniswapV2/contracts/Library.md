# Library

```solidity
pragma solidity >=0.5.0;

import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol';

import "./SafeMath.sol";

library UniswapV2Library {
    using SafeMath for uint;

    // returns sorted token addresses, used to handle return values from pairs sorted in this order
    function sortTokens(address tokenA, address tokenB) internal pure returns (address token0, address token1) {
        require(tokenA != tokenB, 'UniswapV2Library: IDENTICAL_ADDRESSES');
        (token0, token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2Library: ZERO_ADDRESS');
    }

    // calculates the CREATE2 address for a pair without making any external calls
    function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
        (address token0, address token1) = sortTokens(tokenA, tokenB);
        pair = address(uint(keccak256(abi.encodePacked(
                hex'ff',
                factory,
                keccak256(abi.encodePacked(token0, token1)),
                hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
            ))));
    }

    // fetches and sorts the reserves for a pair
    function getReserves(address factory, address tokenA, address tokenB) internal view returns (uint reserveA, uint reserveB) {
        (address token0,) = sortTokens(tokenA, tokenB);
        (uint reserve0, uint reserve1,) = IUniswapV2Pair(pairFor(factory, tokenA, tokenB)).getReserves();
        (reserveA, reserveB) = tokenA == token0 ? (reserve0, reserve1) : (reserve1, reserve0);
    }

    // given some amount of an asset and pair reserves, returns an equivalent amount of the other asset
    function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB) {
        require(amountA > 0, 'UniswapV2Library: INSUFFICIENT_AMOUNT');
        require(reserveA > 0 && reserveB > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        amountB = amountA.mul(reserveB) / reserveA;
    }

    // given an input amount of an asset and pair reserves, returns the maximum output amount of the other asset
    function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
        require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint amountInWithFee = amountIn.mul(997);
        uint numerator = amountInWithFee.mul(reserveOut);
        uint denominator = reserveIn.mul(1000).add(amountInWithFee);
        amountOut = numerator / denominator;
    }

    // given an output amount of an asset and pair reserves, returns a required input amount of the other asset
    function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn) {
        require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint numerator = reserveIn.mul(amountOut).mul(1000);
        uint denominator = reserveOut.sub(amountOut).mul(997);
        amountIn = (numerator / denominator).add(1);
    }

    // performs chained getAmountOut calculations on any number of pairs
    function getAmountsOut(address factory, uint amountIn, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[0] = amountIn;
        for (uint i; i < path.length - 1; i++) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i], path[i + 1]);
            amounts[i + 1] = getAmountOut(amounts[i], reserveIn, reserveOut);
        }
    }

    // performs chained getAmountIn calculations on any number of pairs
    function getAmountsIn(address factory, uint amountOut, address[] memory path) internal view returns (uint[] memory amounts) {
        require(path.length >= 2, 'UniswapV2Library: INVALID_PATH');
        amounts = new uint[](path.length);
        amounts[amounts.length - 1] = amountOut;
        for (uint i = path.length - 1; i > 0; i--) {
            (uint reserveIn, uint reserveOut) = getReserves(factory, path[i - 1], path[i]);
            amounts[i - 1] = getAmountIn(amounts[i], reserveIn, reserveOut);
        }
    }
}
```

# <font style="color:rgb(28, 30, 33);">Internal Functions</font>
## <font style="color:rgb(28, 30, 33);">sortTokens</font>
```solidity
function sortTokens(address tokenA, address tokenB) internal pure returns (address token0, address token1);
```

**<font style="color:rgb(28, 30, 33);">Sorts token addresses.</font>**

## <font style="color:rgb(28, 30, 33);">pairFor</font>
```solidity
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair);
```

**<font style="color:rgb(28, 30, 33);">Calculates the address for a pair without making any external calls via the v2 SDK.</font>**

## <font style="color:rgb(28, 30, 33);">getReserves</font>
```solidity
function getReserves(address factory, address tokenA, address tokenB) internal view returns (uint reserveA, uint reserveB);
```

**<font style="color:rgb(28, 30, 33);">Calls</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getReserves</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#getreserves)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">on the pair for the passed tokens, and returns the results sorted in the order that the parameters were passed in.</font>**

## <font style="color:rgb(28, 30, 33);">quote</font>
```solidity
function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB);
```

**<font style="color:rgb(28, 30, 33);">Given some asset amount and reserves, returns an amount of the other asset representing equivalent value.</font>**

+ <font style="color:rgb(34, 34, 34) !important;">Useful for calculating optimal token amounts before calling</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">mint</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#mint-1)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">getAmountOut</font>
```solidity
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut);
```

**<font style="color:rgb(28, 30, 33);">Given an</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">input</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">asset amount, returns the maximum</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">output</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">amount of the other asset (accounting for fees) given reserves.</font>**

+ <font style="color:rgb(34, 34, 34) !important;">Used in</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">getAmountsOut</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getamountsout)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">getAmountIn</font>
```solidity
function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn);
```

**<font style="color:rgb(28, 30, 33);">Returns the minimum</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">input</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">asset amount required to buy the given</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">output</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">asset amount (accounting for fees) given reserves.</font>**

+ <font style="color:rgb(34, 34, 34) !important;">Used in</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">getAmountsIn</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getamountsin)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">getAmountsOut</font>
```solidity
function getAmountsOut(uint amountIn, address[] memory path) internal view returns (uint[] memory amounts);
```

**<font style="color:rgb(28, 30, 33);">Given an</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">input</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">asset amount and an array of token addresses, calculates all subsequent maximum</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">output</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">token amounts by calling</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getReserves</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getreserves)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">for each pair of token addresses in the path in turn, and using these to call</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getAmountOut</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getamountout)**<font style="color:rgb(28, 30, 33);">.</font>**

+ <font style="color:rgb(34, 34, 34) !important;">Useful for calculating optimal token amounts before calling</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">swap</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#swap-1)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">getAmountsIn</font>
```solidity
function getAmountsIn(address factory, uint amountOut, address[] memory path) internal view returns (uint[] memory amounts);
```

**<font style="color:rgb(28, 30, 33);">Given an</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">output</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">asset amount and an array of token addresses, calculates all preceding minimum</font>****<font style="color:rgb(28, 30, 33);"> </font>**_**<font style="color:rgb(28, 30, 33);">input</font>**_**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">token amounts by calling</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getReserves</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getreserves)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">for each pair of token addresses in the path in turn, and using these to call</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getAmountIn</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/library#getamountin)**<font style="color:rgb(28, 30, 33);">.</font>**

+ <font style="color:rgb(34, 34, 34) !important;">Useful for calculating optimal token amounts before calling </font>[<font style="color:rgb(239, 3, 172);">swap</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#swap-1)<font style="color:rgb(34, 34, 34) !important;">.</font>



> 更新: 2025-09-20 20:40:21  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/fzc7360qogqgp5vn>