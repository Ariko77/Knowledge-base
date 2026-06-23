# PairERC20

```solidity
pragma solidity =0.5.16;

import './interfaces/IUniswapV2ERC20.sol';
import './libraries/SafeMath.sol';

contract UniswapV2ERC20 is IUniswapV2ERC20 {
    using SafeMath for uint;

    string public constant name = 'Uniswap V2';
    string public constant symbol = 'UNI-V2';
    uint8 public constant decimals = 18;
    uint  public totalSupply;
    mapping(address => uint) public balanceOf;
    mapping(address => mapping(address => uint)) public allowance;

    bytes32 public DOMAIN_SEPARATOR;
    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 public constant PERMIT_TYPEHASH = 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;
    mapping(address => uint) public nonces;

    event Approval(address indexed owner, address indexed spender, uint value);
    event Transfer(address indexed from, address indexed to, uint value);

    constructor() public {
        uint chainId;
        assembly {
            chainId := chainid
        }
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
                keccak256(bytes(name)),
                keccak256(bytes('1')),
                chainId,
                address(this)
            )
        );
    }

    function _mint(address to, uint value) internal {
        totalSupply = totalSupply.add(value);
        balanceOf[to] = balanceOf[to].add(value);
        emit Transfer(address(0), to, value);
    }

    function _burn(address from, uint value) internal {
        balanceOf[from] = balanceOf[from].sub(value);
        totalSupply = totalSupply.sub(value);
        emit Transfer(from, address(0), value);
    }

    function _approve(address owner, address spender, uint value) private {
        allowance[owner][spender] = value;
        emit Approval(owner, spender, value);
    }

    function _transfer(address from, address to, uint value) private {
        balanceOf[from] = balanceOf[from].sub(value);
        balanceOf[to] = balanceOf[to].add(value);
        emit Transfer(from, to, value);
    }

    function approve(address spender, uint value) external returns (bool) {
        _approve(msg.sender, spender, value);
        return true;
    }

    function transfer(address to, uint value) external returns (bool) {
        _transfer(msg.sender, to, value);
        return true;
    }

    function transferFrom(address from, address to, uint value) external returns (bool) {
        if (allowance[from][msg.sender] != uint(-1)) {
            allowance[from][msg.sender] = allowance[from][msg.sender].sub(value);
        }
        _transfer(from, to, value);
        return true;
    }

    function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external {
        require(deadline >= block.timestamp, 'UniswapV2: EXPIRED');
        bytes32 digest = keccak256(
            abi.encodePacked(
                '\x19\x01',
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline))
            )
        );
        address recoveredAddress = ecrecover(digest, v, r, s);
        require(recoveredAddress != address(0) && recoveredAddress == owner, 'UniswapV2: INVALID_SIGNATURE');
        _approve(owner, spender, value);
    }
}
```

# <font style="color:rgb(28, 30, 33);">Events</font>

## <font style="color:rgb(28, 30, 33);">Approval</font>

```solidity
event Approval(address indexed owner, address indexed spender, uint value);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time an approval occurs via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">approve</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#approve)**<font style="color:rgb(28, 30, 33);"> </font>****<font style="color:rgb(28, 30, 33);">or</font>****<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">permit</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#permit)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">Transfer</font>

```solidity
event Transfer(address indexed from, address indexed to, uint value);
```

**<font style="color:rgb(28, 30, 33);">Emitted each time a transfer occurs via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">transfer</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transfer-1)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">transferFrom</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transferfrom)**<font style="color:rgb(28, 30, 33);">,</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">mint</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#mint-1)**<font style="color:rgb(28, 30, 33);">, or</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">burn</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair#burn-1)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">Read-Only Functions</font>

## <font style="color:rgb(28, 30, 33);">name</font>

```solidity
function name() external pure returns (string memory);
```

**<font style="color:rgb(28, 30, 33);">Returns</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">Uniswap V2</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">for all pairs.</font>**

## <font style="color:rgb(28, 30, 33);">symbol</font>

```solidity
function symbol() external pure returns (string memory);
```

**<font style="color:rgb(28, 30, 33);">Returns</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">UNI-V2</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">for all pairs.</font>**

## <font style="color:rgb(28, 30, 33);">decimals</font>

```solidity
function decimals() external pure returns (uint8);
```

**<font style="color:rgb(28, 30, 33);">Returns</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">18</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">for all pairs.</font>**

## <font style="color:rgb(28, 30, 33);">totalSupply</font>

```solidity
function totalSupply() external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the total amount of pool tokens for a pair.</font>**

## <font style="color:rgb(28, 30, 33);">balanceOf</font>

```solidity
function balanceOf(address owner) external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the amount of pool tokens owned by an address.</font>**

## <font style="color:rgb(28, 30, 33);">allowance</font>

```solidity
function allowance(address owner, address spender) external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the amount of liquidity tokens owned by an address that a spender is allowed to transfer via</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">transferFrom</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transferfrom)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">DOMAIN\_SEPARATOR</font>

```solidity
function DOMAIN_SEPARATOR() external view returns (bytes32);
```

**<font style="color:rgb(28, 30, 33);">Returns a domain separator for use in</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">permit</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#permit)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">PERMIT\_TYPEHASH</font>

```solidity
function PERMIT_TYPEHASH() external view returns (bytes32);
```

**<font style="color:rgb(28, 30, 33);">Returns a typehash for use in</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">permit</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#permit)**<font style="color:rgb(28, 30, 33);">.</font>**

## <font style="color:rgb(28, 30, 33);">nonces</font>

```solidity
function nonces(address owner) external view returns (uint);
```

**<font style="color:rgb(28, 30, 33);">Returns the current nonce for an address for use in</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**[**<font style="color:rgb(239, 3, 172);">permit</font>**](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#permit)**<font style="color:rgb(28, 30, 33);">.</font>**

# <font style="color:rgb(28, 30, 33);">State-Changing Functions</font>

## <font style="color:rgb(28, 30, 33);">approve</font>

```solidity
function approve(address spender, uint value) external returns (bool);
```

**<font style="color:rgb(28, 30, 33);">Lets</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">msg.sender</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">set their allowance for a spender.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Approval</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#approval)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">transfer</font>

```solidity
function transfer(address to, uint value) external returns (bool);
```

**<font style="color:rgb(28, 30, 33);">Lets</font>\*\*\*\*<font style="color:rgb(28, 30, 33);"> </font>**<code>**<font style="color:rgb(28, 30, 33);background-color:rgb(246, 247, 248);">msg.sender</font>**</code>**<font style="color:rgb(28, 30, 33);"> </font>\*\*\*\*<font style="color:rgb(28, 30, 33);">send pool tokens to an address.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Transfer</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transfer)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">transferFrom</font>

```solidity
function transferFrom(address from, address to, uint value) external returns (bool);
```

**<font style="color:rgb(28, 30, 33);">Sends pool tokens from one address to another.</font>**

* <font style="color:rgb(34, 34, 34) !important;">Requires approval.</font>
* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Transfer</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#transfer)<font style="color:rgb(34, 34, 34) !important;">.</font>

## <font style="color:rgb(28, 30, 33);">permit</font>

```solidity
function permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s) external;
```

**<font style="color:rgb(28, 30, 33);">Sets the allowance for a spender where approval is granted via a signature.</font>**

* <font style="color:rgb(34, 34, 34) !important;">See</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Using Permit</font>](https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/supporting-meta-transactions)<font style="color:rgb(34, 34, 34) !important;">.</font>
* <font style="color:rgb(34, 34, 34) !important;">Emits</font><font style="color:rgb(34, 34, 34) !important;"> </font>[<font style="color:rgb(239, 3, 172);">Approval</font>](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair-erc-20#approval)<font style="color:rgb(34, 34, 34) !important;">.</font>

# <font style="color:rgb(28, 30, 33);">Interface</font>

```solidity
import '@uniswap/v2-core/contracts/interfaces/IUniswapV2ERC20.sol';
```

```solidity
pragma solidity >=0.5.0;

interface IUniswapV2ERC20 {
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
}
```

# <font style="color:rgb(28, 30, 33);">ABI</font>

```typescript
import IUniswapV2ERC20 from '@uniswap/v2-core/build/IUniswapV2ERC20.json'
```

[**<font style="color:rgb(239, 3, 172);">https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2ERC20.json</font>**](https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2ERC20.json)


> 更新: 2025-09-20 20:29:12  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/xgp5yyb06gn7wza3>