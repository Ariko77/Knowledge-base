# Pair

# Address

## <font style="color:rgb(28, 30, 33);">getPair</font>

**<font style="color:rgb(28, 30, 33);">The most obvious way to get the address for a pair is to call</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">getPair</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#getpair)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">on the factory. If the pair exists, this function will return its address, else</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">address(0)</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">(</font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0x0000000000000000000000000000000000000000</font>**</code>**<font style="color:rgb(28, 30, 33);">).</font>**

* <font style="color:rgb(34, 34, 34) !important;">The "canonical" way to determine whether or not a pair exists.</font>
* <font style="color:rgb(34, 34, 34) !important;">Requires an on-chain lookup.</font>

## <font style="color:rgb(28, 30, 33);">CREATE2</font>

**<font style="color:rgb(28, 30, 33);">Thanks to some</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">fancy footwork in the factory</font>**](https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2Factory.sol#L32)**<font style="color:rgb(28, 30, 33);">, we can also compute pair addresses</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>*****<font style="color:rgb(28, 30, 33);">without any on-chain lookups</font>*****<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">because of</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">CREATE2</font>**](https://eips.ethereum.org/EIPS/eip-1014)**<font style="color:rgb(28, 30, 33);">. The following values are required for this technique:</font>**

| | |
| --- | --- |
| <code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">address</font>**</code> | **<font style="color:rgb(28, 30, 33);">The</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">factory address</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#address) |
| <code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">salt</font>**</code> | <code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">keccak256(abi.encodePacked(token0, token1))</font>**</code> |
| <code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">keccak256(init_code)</font>**</code> | <code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0x96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f</font>**</code> |

* <code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">token0</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">must be strictly less than</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">token1</font></code><font style="color:rgb(34, 34, 34) !important;"> </font><font style="color:rgb(34, 34, 34) !important;">by sort order.</font>
* <font style="color:rgb(34, 34, 34) !important;">Can be computed offline.</font>
* <font style="color:rgb(34, 34, 34) !important;">Requires the ability to perform</font><font style="color:rgb(34, 34, 34) !important;"> </font><code><font style="color:rgb(34, 34, 34) !important;background-color:rgb(246, 247, 248);">keccak256</font></code><font style="color:rgb(34, 34, 34) !important;">.</font>

### <font style="color:rgb(28, 30, 33);">Examples</font>

```solidity
address factory = 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
address token0 = 0xCAFE000000000000000000000000000000000000; // change me!
address token1 = 0xF00D000000000000000000000000000000000000; // change me!

address pair = address(uint160(bytes20(keccak256(abi.encodePacked(
  hex'ff',
  factory,
  keccak256(abi.encodePacked(token0, token1)),
  hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f'
))));
```

# <font style="color:rgb(28, 30, 33);">Events</font>

## <font style="color:rgb(28, 30, 33);">Mint</font>

```solidity
event Mint(address indexed sender, uint amount0, uint amount1);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time liquidity tokens are created via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">mint</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#mint-1)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">Burn</font>

```solidity
event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time liquidity tokens are destroyed via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">burn</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#burn-1)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">Swap</font>

```solidity
event Swap(
  address indexed sender,
  uint amount0In,
  uint amount1In,
  uint amount0Out,
  uint amount1Out,
  address indexed to
);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time a swap occurs via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">swap</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#swap-1)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">Sync</font>

```solidity
event Sync(uint112 reserve0, uint112 reserve1);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time reserves are updated via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">mint</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#mint-1)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">burn</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#burn-1)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">swap</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#swap-1)**<font style="color:rgb(28, 30, 33);">, or</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">sync</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#sync-1)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">Read-Only Functions</font>

## <font style="color:rgb(28, 30, 33);">MINIMUM\_LIQUIDITY</font>

```solidity
function MINIMUM_LIQUIDITY() external pure returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">1000</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">for all pairs. See</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Minimum Liquidity</font>**](https://docs.uniswap.org/contracts/v2/concepts/protocol-overview/smart-contracts#minimum-liquidity)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">factory</font>

```solidity
function factory() external view returns (address);
```

**<font style="color:rgb(28, 30, 33);">Returns the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">factory address</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory#address)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">token0</font>

```solidity
function token0() external view returns (address);
```

**<font style="color:rgb(28, 30, 33);">Returns the address of the pair token with the lower sort order.</font>**

## <font style="color:rgb(28, 30, 33);">token1</font>

```solidity
function token1() external view returns (address);
```

**<font style="color:rgb(28, 30, 33);">Returns the address of the pair token with the higher sort order.</font>**

## <font style="color:rgb(28, 30, 33);">getReserves</font>

```solidity
function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
```

**<font style="color:rgb(28, 30, 33);">Returns the reserves of token0 and token1 used to price trades and distribute liquidity. See</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Pricing</font>**](https://docs.uniswap.org/contracts/v2/concepts/advanced-topics/pricing)**<font style="color:rgb(28, 30, 33);">. Also returns the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">block.timestamp</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">(mod</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">2**32</font>**</code>**<font style="color:rgb(28, 30, 33);">) of the last block during which an interaction occured for the pair.</font>**

## <font style="color:rgb(28, 30, 33);">price0CumulativeLast</font>

```solidity
function price0CumulativeLast() external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">See</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Oracles</font>**](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">price1CumulativeLast</font>

```solidity
function price1CumulativeLast() external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">See</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Oracles</font>**](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">kLast</font>

```solidity
function kLast() external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the product of the reserves as of the most recent liquidity event. See</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Protocol Charge Calculation</font>**](https://docs.uniswap.org/contracts/v2/concepts/advanced-topics/fees#protocol-charge-calculation)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">State-Changing Functions</font>

## <font style="color:rgb(28, 30, 33);">mint</font>

```solidity
function mint(address to) external returns (uint liquidity);
```

**<font style="color:rgb(28, 30, 33);">Creates pool tokens.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Mint</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#mint)<font style="color:rgb(34, 34, 34) !important;">,</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Sync</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#sync)<font style="color:rgb(34, 34, 34) !important;">,</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Transfer</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transfer)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">burn</font>

```solidity
function burn(address to) external returns (uint amount0, uint amount1);
```

**<font style="color:rgb(28, 30, 33);">Destroys pool tokens.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Burn</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#burn)<font style="color:rgb(34, 34, 34) !important;">,</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Sync</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#sync)<font style="color:rgb(34, 34, 34) !important;">,</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Transfer</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transfer)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">swap</font>

```solidity
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
```

**<font style="color:rgb(28, 30, 33);">Swaps tokens. For regular swaps,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">data.length</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">must be</font>****<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">0</font>**</code>**<font style="color:rgb(28, 30, 33);">. Also see</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">Flash Swaps</font>**](https://docs.uniswap.org/contracts/v2/concepts/core-concepts/flash-swaps)**<font style="color:rgb(28, 30, 33);">.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Swap</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#swap)<font style="color:rgb(34, 34, 34) !important;">,</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Sync</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#sync)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">skim</font>

```solidity
function skim(address to) external;
```

**<font style="color:rgb(28, 30, 33);">See the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">whitepaper</font>**](https://docs.uniswap.org/whitepaper.pdf)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">sync</font>

```solidity
function sync() external;
```

**<font style="color:rgb(28, 30, 33);">See the</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">whitepaper</font>**](https://docs.uniswap.org/whitepaper.pdf)**<font style="color:rgb(28, 30, 33);">.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Sync</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#sync)<font style="color:rgb(34, 34, 34) !important;">.</font>

# <font style="color:rgb(28, 30, 33);">Interface</font>

```solidity
import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol';
```

```solidity
pragma solidity >=0.5.0;

interface IUniswapV2Pair {
  event Approval(address indexed owner, address indexed spender, uint value);
  event Transfer(address indexed from, address indexed to, uint value);

  function name() external pure returns (string memory);
  function symbol() external pure returns (string memory);
  function decimals() external pure returns (uint8);
  function totalSupply() external view returns (uint);
  function balanceOf(address owner) external view returns (uint);
  function allowance(address owner, address spender) external view returns (uint);

  function approve(address spender, uint value) external returns (bool);
  function transfer(address to, uint value) external returns (bool);
  function transferFrom(address from, address to, uint value) external returns (bool);

  function DOMAIN_SEPARATOR() external view returns (bytes32);
  function PERMIT_TYPEHASH() external pure returns (bytes32);
  function nonces(address owner) external view returns (uint);

  function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external;

  event Mint(address indexed sender, uint amount0, uint amount1);
  event Burn(address indexed sender, uint amount0, uint amount1, address indexed to);
  event Swap(
      address indexed sender,
      uint amount0In,
      uint amount1In,
      uint amount0Out,
      uint amount1Out,
      address indexed to
  );
  event Sync(uint112 reserve0, uint112 reserve1);

  function MINIMUM_LIQUIDITY() external pure returns (uint);
  function factory() external view returns (address);
  function token0() external view returns (address);
  function token1() external view returns (address);
  function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
  function price0CumulativeLast() external view returns (uint);
  function price1CumulativeLast() external view returns (uint);
  function kLast() external view returns (uint);

  function mint(address to) external returns (uint liquidity);
  function burn(address to) external returns (uint amount0, uint amount1);
  function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
  function skim(address to) external;
  function sync() external;
}
```

# <font style="color:rgb(28, 30, 33);">ABI</font>

```typescript
import IUniswapV2Pair from '@uniswap/v2-core/build/IUniswapV2Pair.json'
```

[**<font style="color:rgb(239, 3, 172);">https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Pair.json</font>**](https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Pair.json)


> 更新: 2025-09-20 20:20:47  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/uc30wwzhccu86f21>