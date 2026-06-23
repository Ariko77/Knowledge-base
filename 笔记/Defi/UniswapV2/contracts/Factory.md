# Factory

```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2Factory.sol';
import './UniswapV2Pair.sol';

contract UniswapV2Factory is IUniswapV2Factory {
    address public feeTo;
    address public feeToSetter;

    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    event PairCreated(address indexed token0, address indexed token1, address pair, uint);

    constructor(address _feeToSetter) public {
        feeToSetter = _feeToSetter;
    }

    function allPairsLength() external view returns (uint) {
        return allPairs.length;
    }

    function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        IUniswapV2Pair(pair).initialize(token0, token1);
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }

    function setFeeTo(address _feeTo) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeTo = _feeTo;
    }

    function setFeeToSetter(address _feeToSetter) external {
        require(msg.sender == feeToSetter, 'UniswapV2: FORBIDDEN');
        feeToSetter = _feeToSetter;
    }
}
```

# <font style="color:rgb(28, 30, 33);">Address</font>

<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">UniswapV2Factory</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">is deployed at</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">on the Ethereum</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">mainnet</font>**](https://etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)**<font style="color:rgb(28, 30, 33);">, and the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Ropsten</font>**](https://ropsten.etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Rinkeby</font>**](https://rinkeby.etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Görli</font>**](https://goerli.etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)**<font style="color:rgb(28, 30, 33);">, and</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Kovan</font>**](https://kovan.etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">testnets. It was built from commit</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">8160750</font>**](https://github.com/Uniswap/uniswap-v2-core/tree/816075049f811f1b061bca81d5d040b96f4c07eb)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">Events</font>

## <font style="color:rgb(28, 30, 33);">PairCreated</font>

```solidity
event PairCreated(address indexed token0, address indexed token1, address pair, uint);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time a pair is created via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">createPair</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#createpair)**<font style="color:rgb(28, 30, 33);">.</font>**

* <code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">token0</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">is guaranteed to be strictly less than</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">token1</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">by sort order.</font>
* <font style="color:rgb(34, 34, 34) !important;">The final</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">uint</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">log value will be</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">1</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">for the first pair created,</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">2</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">for the second, etc. (see</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">allPairs</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#allpairs)<font style="color:rgb(34, 34, 34) !important;">/</font>[<font style="color:rgb(239, 3, 172);">getPair</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#getpair)<font style="color:rgb(34, 34, 34) !important;">).</font>

# <font style="color:rgb(28, 30, 33);">Read-Only Functions</font>

## <font style="color:rgb(28, 30, 33);">getPair</font>

```solidity
function getPair(address tokenA, address tokenB) external view returns (address pair);
```

**<font style="color:rgb(28, 30, 33);">Returns the address of the pair for</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">tokenA</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">and</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">tokenB</font>**</code>**<font style="color:rgb(28, 30, 33);">, if it has been created, else</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">address(0)</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">(</font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0x0000000000000000000000000000000000000000</font>**</code>**<font style="color:rgb(28, 30, 33);">).</font>**

* <code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">tokenA</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">and</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">tokenB</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">are interchangeable.</font>
* <font style="color:rgb(34, 34, 34) !important;">Pair addresses can also be calculated deterministically via the SDK.</font>

## <font style="color:rgb(28, 30, 33);">allPairs</font>

```solidity
function allPairs(uint) external view returns (address pair);
```

**<font style="color:rgb(28, 30, 33);">Returns the address of the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">n</font>**</code>**<font style="color:rgb(28, 30, 33);">th pair (</font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0</font>**</code>**<font style="color:rgb(28, 30, 33);">-indexed) created through the factory, or</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">address(0)</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">(</font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0x0000000000000000000000000000000000000000</font>**</code>**<font style="color:rgb(28, 30, 33);">) if not enough pairs have been created yet.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Pass</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">0</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">for the address of the first pair created,</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">1</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">for the second, etc.</font>

## <font style="color:rgb(28, 30, 33);">allPairsLength</font>

```solidity
function allPairsLength() external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the total number of pairs created through the factory so far.</font>**

## <font style="color:rgb(28, 30, 33);">feeTo</font>

```solidity
function feeTo() external view returns (address);
```

**<font style="color:rgb(28, 30, 33);">See</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Protocol Charge Calculation</font>**](https://docs.uniswap.org/contracts/v2/concepts/advanced-topics/fees)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">feeToSetter</font>

```solidity
function feeToSetter() external view returns (address);
```

**<font style="color:rgb(28, 30, 33);">The address allowed to change</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">feeTo</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#feeto)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">State-Changing Functions</font>

## <font style="color:rgb(28, 30, 33);">createPair</font>

```solidity
function createPair(address tokenA, address tokenB) external returns (address pair);
```

**<font style="color:rgb(28, 30, 33);">Creates a pair for</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">tokenA</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">and</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">tokenB</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">if one doesn't exist already.</font>**

* <code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">tokenA</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">and</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">tokenB</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">are interchangeable.</font>
* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">PairCreated</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#paircreated)<font style="color:rgb(34, 34, 34) !important;">.</font>

# <font style="color:rgb(28, 30, 33);">Interface</font>

```solidity
import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol';
```

```solidity
pragma solidity >=0.5.0;

interface IUniswapV2Factory {
  event PairCreated(address indexed token0, address indexed token1, address pair, uint);

  function getPair(address tokenA, address tokenB) external view returns (address pair);
  function allPairs(uint) external view returns (address pair);
  function allPairsLength() external view returns (uint);

  function feeTo() external view returns (address);
  function feeToSetter() external view returns (address);

  function createPair(address tokenA, address tokenB) external returns (address pair);
}
```

# <font style="color:rgb(28, 30, 33);">ABI</font>

```typescript
import IUniswapV2Factory from '@uniswap/v2-core/build/IUniswapV2Factory.json'
```

[**<font style="color:rgb(239, 3, 172);">https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Factory.json</font>**](https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Factory.json)


> 更新: 2025-09-20 20:18:52  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ti0zaccg0exn77c7>