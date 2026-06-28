# Fallout 

# Fallout

## 合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

目标是成为owner

## 思路分析

观察可得这个**Fal\*\*\*\*<font style="background-color:#f3bb2f;">1</font>\*\*\*\*out**其实是一个函数

```solidity
function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
```

直接调用就可以成为ownner

![](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\image-20250416215518868.png)

成功

## 一些用到的知识

### .env

可以将私钥,SEPOLIA\_RPC等存在.env里

在终端输入`source .env`即可 注意中间有个空格不要忘记

使用方法就如上图所示,比如用`$SEPOLIA_RPC`代替平时要打在终端里的即可

### Alchemy 提供的 RPC 地址

[Alchemy官网](https://dashboard.alchemy.com/apps/pbyu8wbl1802y71x/setup)

这个其中就包含了apikey,可以写进.env来使用


> 更新: 2025-07-09 11:27:28  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ugm410a080cmp21b>